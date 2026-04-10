
S3 Tables é a camada de armazenamento tabular nativa da AWS, construída sobre o formato Apache Iceberg. Representa a aposta da AWS na convergência do conceito de **Lakehouse**: manter dados brutos em object storage (custo baixo, escala ilimitada) com semântica transacional de banco de dados (ACID, schema evolution, time travel).

É o componente central do **Amazon SageMaker Lakehouse** — a plataforma unificada que a AWS está construindo para substituir arquiteturas fragmentadas de Data Lake + Data Warehouse.

## Índice

- [[#Por que S3 Tables existe]]
- [[#Apache Iceberg — O Formato por Baixo]]
- [[#Arquitetura — Table Buckets]]
- [[#Manutenção Automática de Tabelas]]
- [[#Performance]]
- [[#S3 Tables Intelligent-Tiering]]
- [[#Replicação]]
- [[#Segurança]]
- [[#Integração com o Ecossistema AWS]]
- [[#S3 Tables vs S3 Buckets Padrão]]
- [[#Contexto em Data Lakes]]

---

## Por que S3 Tables existe

### O problema com S3 Buckets tradicionais para analytics

S3 padrão armazena **objetos isolados** — não tem conceito de tabela, transação, nem schema. Para usar S3 como base de um Data Lake analítico, times precisavam resolver manualmente:

- **Small files problem**: writes frequentes geram milhares de arquivos pequenos → queries lentas (overhead de LIST + GET por arquivo)
- **Gestão de snapshots**: cada modificação via Iceberg cria um novo snapshot → acúmulo infinito de metadados sem limpeza automática
- **Compaction manual**: agrupar arquivos pequenos em arquivos maiores exige jobs periódicos (Spark, EMR) que precisam ser orquestrados
- **Consistência**: sem isolamento transacional, leituras e escritas concorrentes causam leituras sujas (dirty reads)

S3 Tables resolve tudo isso de forma **gerenciada e automática**.

---

## Apache Iceberg — O Formato por Baixo

Apache Iceberg é um **formato de tabela aberto** para grandes datasets analíticos. Não é um banco de dados nem um motor de query — é uma especificação de como organizar arquivos e metadados para que diferentes engines (Spark, Flink, Trino, Athena) enxerguem os dados como uma tabela com semântica relacional.

### Estrutura de metadados do Iceberg

```
Catalog (ex: S3 Tables REST Catalog)
  └── Namespace (equivalente a "database")
        └── Table
              └── Metadata File (JSON) — aponta para o snapshot atual
                    └── Snapshot
                          └── Manifest List (lista de manifest files)
                                └── Manifest File (lista de data files + estatísticas)
                                      └── Data Files (Parquet, ORC, Avro)
```

Essa hierarquia de metadados é o que permite as features avançadas do Iceberg:

### ACID Compliance

| Propriedade | Significado | Benefício |
|---|---|---|
| **Atomicity** | Operação inteira falha ou tem sucesso — sem estado parcial | Não há dados corrompidos por falhas de job |
| **Consistency** | Cada transação leva o banco de um estado válido para outro | Invariantes de schema sempre respeitados |
| **Isolation** | Leituras e escritas concorrentes não se interferem | Queries não leem dados "em meio" a um write |
| **Durability** | Uma vez confirmada, a transação persiste | Dados não se perdem mesmo com falha após o commit |

> Antes do Iceberg, o S3 era apenas eventually consistent (resolvido em 2020 com strong consistency). Mas ter read-after-write não é suficiente para analytics — você precisa de isolamento entre transações concorrentes, que o Iceberg entrega.

### Schema Evolution

Alterar o schema de uma tabela sem reescrever os dados históricos:

| Operação | Suportada? | Comportamento |
|---|---|---|
| Adicionar coluna | Sim | Dados antigos retornam `null` na nova coluna |
| Remover coluna | Sim | Dados antigos continuam acessíveis por ID da coluna |
| Renomear coluna | Sim | O mapping ID→nome é preservado nos metadados |
| Reordenar colunas | Sim | A ordem lógica não afeta o layout físico |
| Mudar tipo de coluna | Parcial | Widening (int→long) sim; narrowing não |

> O Iceberg usa **IDs internos** para referenciar colunas, não nomes. Por isso renames e reorders não quebram leitores que usam versões antigas do schema.

### Particionamento Oculto (Hidden Partitioning)

No Hive (modelo antigo), o usuário precisa escrever `WHERE year = 2024 AND month = 4` para aproveitar partições. No Iceberg:

- As partições são definidas na **spec da tabela** (ex: particionado por `month(event_time)`)
- O engine **resolve o filtro automaticamente** — o usuário escreve `WHERE event_time BETWEEN '2024-04-01' AND '2024-04-30'` e o Iceberg faz o pruning
- A **partition spec pode evoluir** sem reescrever dados: você pode mudar de `month()` para `day()` e ambas as specs coexistem nos metadados

### Time Travel (Viagem no Tempo)

Cada operação de escrita cria um **novo snapshot** imutável. Isso permite:

```sql
-- Consultar o estado da tabela em um momento específico
SELECT * FROM minha_tabela FOR SYSTEM_TIME AS OF '2024-03-15 12:00:00';

-- Consultar por snapshot ID
SELECT * FROM minha_tabela FOR VERSION AS OF 123456789;

-- Comparar versões (ex: auditoria de mudanças)
SELECT * FROM minha_tabela VERSION AS OF 2
EXCEPT
SELECT * FROM minha_tabela VERSION AS OF 1;
```

**Casos de uso**: rollback de dados incorretos ingeridos, auditorias regulatórias, debugging de pipelines, testes reproduzíveis com snapshot fixo.

### Compatibilidade de Engines

Por ser um formato aberto com especificação pública, o Iceberg é suportado por:

| Engine | Suporte |
|---|---|
| Apache Spark | Leitura e escrita (biblioteca `iceberg-spark`) |
| Apache Flink | Leitura e escrita (streaming incremental) |
| Trino / Presto | Leitura e escrita |
| Apache Hive | Leitura |
| Amazon Athena | Leitura e escrita nativa |
| Amazon EMR | Spark/Flink/Hive/Trino — todos |
| Amazon Redshift Spectrum | Leitura |
| Snowflake | Leitura via Iceberg Tables |
| Databricks | Leitura e escrita |

> Isso é o que faz o S3 Tables ser um **open format storage** — não há lock-in de engine. O dado fica em Parquet no S3, os metadados em Iceberg, e qualquer engine que suporte o protocolo Iceberg REST Catalog consegue ler.

![[Pasted image 20260410134405.png]]

---

## Arquitetura — Table Buckets

Table Bucket é o recurso de nível superior do S3 Tables — **não é um S3 Bucket comum**. Ele opera em um namespace de serviço separado (`s3tables`, não `s3`), com suas próprias APIs e modelo de controle de acesso.

```
Table Bucket  (equivale a um "servidor de banco de dados")
  └── Namespace  (equivale a um "database" / "schema")
        └── Table  (a tabela Iceberg propriamente dita)
              ├── Data files  (Parquet)
              └── Metadata files  (JSON de snapshots, manifests)
```

### Namespace

- Agrupa tabelas relacionadas — mesmo conceito de `database` no Glue Data Catalog
- Necessário para integração com AWS Glue e Lake Formation
- Nome deve ser **lowercase** (letras maiúsculas quebram a integração com Glue)
- **Não pode ser renomeado** após criação
- Suporta tags para organização e controle de custo

### Tipos de Table Buckets

| Tipo | Quem gerencia | Exemplos |
|---|---|---|
| **Gerenciado pelo usuário** | Você cria e controla | Tabelas de negócio, Data Lake customizado |
| **Gerenciado pela AWS** | AWS cria automaticamente | S3 Metadata Tables, S3 Inventory Tables |

> As tabelas gerenciadas pela AWS são criadas automaticamente quando você habilita S3 Metadata ou S3 Inventory em um bucket S3 padrão — os resultados são entregues como tabelas Iceberg consultáveis via Athena.

### Regras de nomeação

- Nomes de Table Buckets precisam ser **únicos apenas dentro da sua conta** (não globalmente, como S3 padrão)
- Devem ser lowercase
- Não podem ser renomeados após criação
- As mesmas restrições se aplicam a namespaces

### Iceberg REST Catalog

S3 Tables expõe um **Iceberg REST Catalog** integrado — o ponto de entrada para qualquer engine conectar. Em vez de configurar um Glue Catalog ou um Hive Metastore, você aponta o Spark/Flink/Trino para o endpoint REST do S3 Tables e ele resolve tabelas automaticamente.

![[Pasted image 20260410134657.png]]

---

## Manutenção Automática de Tabelas

Essa é a diferença central entre S3 Tables e gerenciar Iceberg manualmente no S3 padrão: **o S3 Tables executa as tarefas de manutenção automaticamente**, sem que você precise orquestrar jobs.

### Compaction

**O problema**: pipelines de streaming ou escritas frequentes geram muitos arquivos pequenos. Uma tabela com 1 milhão de arquivos de 1 KB é muito mais lenta de consultar do que uma tabela com 100 arquivos de 10 MB — o custo de listar e abrir cada arquivo domina o tempo de query.

**A solução**: Compaction é o processo de mesclar múltiplos arquivos pequenos em arquivos maiores e otimizados para leitura. No S3 padrão, você precisaria rodar um job Spark periodicamente. No S3 Tables, o serviço faz isso automaticamente em background, sem interromper leituras ou escritas.

```
Antes da compaction:
  data/ ├── 0001.parquet (50 KB)
        ├── 0002.parquet (80 KB)
        ├── 0003.parquet (30 KB)
        └── ... (10.000 arquivos)

Após a compaction:
  data/ ├── 0001-compacted.parquet (128 MB)
        └── 0002-compacted.parquet (128 MB)
```

### Snapshot Management

Cada operação de escrita no Iceberg cria um novo snapshot. Sem limpeza, metadados acumulam indefinidamente. S3 Tables mantém automaticamente apenas os snapshots dentro da janela de retenção configurada, removendo snapshots expirados e seus manifests.

### Unreferenced File Removal

Durante escritas com falha ou após compaction, ficam arquivos órfãos no storage — data files que nenhum snapshot referencia. S3 Tables os identifica e remove automaticamente, evitando acúmulo de lixo e custo desnecessário.

---

## Performance

| Métrica | S3 Tables | S3 Bucket padrão com Iceberg manual |
|---|---|---|
| Performance de query | **Até 3x mais rápida** | Baseline |
| Transações por segundo | **Até 10x mais** | Baseline |
| Gestão de manutenção | Automática | Manual (jobs periódicos) |

### Por que 3x mais rápido?

A compaction automática garante que arquivos sempre estejam em tamanho ótimo. Engines como Athena leem menos arquivos e fazem menos chamadas ao S3 — o principal gargalo em queries sobre object storage.

### Por que 10x mais transações?

O modelo de metadados do S3 Tables é otimizado para alta concorrência de escritas. No Iceberg manual no S3, conflitos de commit em alta concorrência requerem retries. O S3 Tables resolve isso com controle de concorrência otimista gerenciado pelo serviço.

![[Pasted image 20260410134506.png]]

---

## S3 Tables Intelligent-Tiering

Feature de otimização de custo que move dados automaticamente entre tiers com base no padrão de acesso — o mesmo conceito do S3 Intelligent-Tiering, mas adaptado para dados tabulares.

| Tier | Ativação | Tipo de acesso |
|---|---|---|
| **Frequent Access (Standard)** | Default para dados novos | Acesso frequente |
| **Infrequent Access** | Sem acesso por **30 dias** | Acesso ocasional |
| **Archive Instant Access** | Sem acesso por **90 dias** | Acesso raro, mas recuperação imediata |

> A diferença para o S3 Intelligent-Tiering padrão é que o tiering aqui opera na granularidade de **tabelas**, não de objetos individuais. O serviço considera padrão de acesso à tabela como um todo.

---

## Replicação

S3 Tables suporta replicação de Table Buckets entre regiões e/ou contas, mantendo consistência do estado Iceberg (snapshots, manifests e data files) no destino.

**Características**:
- A replicação copia tanto os **data files** quanto os **metadados Iceberg** — o destino é uma tabela Iceberg completa e consultável, não apenas uma cópia de arquivos
- Replicação é **assíncrona**
- Requer IAM Role com permissões adequadas em origem e destino
- Suporta replicação **cross-account**

**Casos de uso**:
- Disaster Recovery com RPO baixo para dados analíticos críticos
- Disponibilizar dados para times em outras regiões sem mover a tabela principal
- Isolamento de workloads: tabela de produção em uma conta, cópia para analytics em outra

![[Pasted image 20260410143541.png]]

---

## Segurança

S3 Tables opera como um **serviço distinto do S3 padrão** (`s3tables:*` vs `s3:*`). Isso tem implicações importantes no modelo de segurança:

### IAM e SCPs

As actions do IAM para S3 Tables têm prefixo `s3tables:` — completamente separadas das permissões `s3:*`.

```json
{
  "Effect": "Allow",
  "Action": [
    "s3tables:CreateTableBucket",
    "s3tables:GetTable",
    "s3tables:PutTableData",
    "s3tables:GetTableData"
  ],
  "Resource": "arn:aws:s3tables:us-east-1:123456789:bucket/meu-table-bucket"
}
```

> **Implicação**: políticas `s3:*` existentes **não concedem acesso** a S3 Tables. Times que migram workloads precisam atualizar SCPs e IAM policies explicitamente.

### Service Control Policies (SCPs)

SCPs no AWS Organizations funcionam normalmente com o prefixo `s3tables:*`, permitindo governança centralizada de quem pode criar ou acessar Table Buckets em toda a organização.

### Integração com Lake Formation

Quando o namespace usa lowercase (obrigatório), o S3 Tables se integra com **AWS Lake Formation** para controle de acesso em nível de coluna e linha — o mesmo modelo de governança usado no Glue Data Catalog.

### Criptografia

- Dados armazenados em Parquet no S3 internamente — herda o modelo de criptografia do S3
- Suporta **SSE-S3** (padrão) e **SSE-KMS** (com CMK gerenciada pelo cliente)
- Em trânsito: TLS obrigatório para todas as APIs

![[Pasted image 20260410144340.png]]

---

## Integração com o Ecossistema AWS

### Amazon SageMaker Lakehouse

A interface preferencial para acessar S3 Tables via console e construir pipelines end-to-end. O Lakehouse unifica S3 Tables, Redshift e SageMaker em um único plano de dados.

### Amazon Athena

Consulta S3 Tables diretamente via SQL sem configuração adicional — Athena usa o Iceberg REST Catalog do S3 Tables automaticamente quando a tabela está registrada no Glue.

```sql
-- Athena já reconhece a tabela via Glue
SELECT categoria, SUM(valor) as total
FROM meu_namespace.vendas
WHERE data_venda >= DATE '2024-01-01'
GROUP BY categoria;
```

### Amazon EMR

Clusters EMR com Spark, Flink ou Trino acessam S3 Tables configurando o catalog REST:

```python
spark = SparkSession.builder \
    .config("spark.sql.catalog.s3tables", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.s3tables.catalog-impl", "software.amazon.s3tables.iceberg.S3TablesCatalog") \
    .config("spark.sql.catalog.s3tables.warehouse", "arn:aws:s3tables:us-east-1:123456789:bucket/meu-table-bucket") \
    .getOrCreate()
```

### AWS Glue

Glue Data Catalog serve como **ponto de registro** — as tabelas do S3 Tables aparecem no Glue como tabelas Iceberg, consultáveis por Athena, EMR e Redshift Spectrum.

---

## S3 Tables vs S3 Buckets Padrão

| Critério | S3 Bucket padrão | S3 Tables |
|---|---|---|
| **Tipo de dado** | Qualquer objeto (imagens, logs, JSON, CSV...) | Dados tabulares estruturados |
| **Formato** | Qualquer formato | Apache Iceberg (Parquet interno) |
| **ACID** | Não | Sim |
| **Compaction** | Manual (job Spark/EMR) | Automática |
| **Schema evolution** | Não gerenciado | Gerenciado pelo Iceberg |
| **Time travel** | Não | Sim (via snapshots Iceberg) |
| **Custo** | Mais barato | Mais caro (serviço gerenciado + storage) |
| **Performance de query** | Depende de otimização manual | Otimizada automaticamente |
| **Casos de uso** | Armazenamento geral, staging, arquivos não estruturados | Tabelas analíticas de alta frequência de escrita |

> **Regra prática**: se você vai fazer queries SQL frequentes e escritas contínuas no dado, S3 Tables tende a ser mais econômico no total (menos custo operacional de manutenção, menos tempo de query, menos custo de compute). Para dados não tabulares ou arquivamento frio, S3 padrão é mais adequado.

---

## Contexto em Data Lakes

S3 Tables se posiciona principalmente nas camadas **intermediárias e finais** de um Data Lake moderno:

| Zona | Recomendação | Justificativa |
|---|---|---|
| **Landing / Raw** | S3 Bucket padrão | Dados chegam em formatos variados (JSON, CSV, binário); S3 Tables exige Iceberg |
| **Trusted / Curated** | **S3 Tables** | Alta frequência de escritas incrementais, necessidade de ACID, schema evolution |
| **Refined / Serving** | **S3 Tables** ou Redshift | Leituras frequentes por BI — compaction automática maximiza performance |
| **Compliance / Archive** | S3 Glacier ou S3 Tables + Intelligent-Tiering | Retenção longa, acesso raro |

### Padrão de pipeline com S3 Tables

```
Fontes de dados
  → S3 Bucket (Landing/Raw)   ← ingestão de qualquer formato
      → Glue ETL / EMR Spark  ← transformação e validação
          → S3 Tables (Trusted/Refined)  ← storage analítico com ACID
              → Athena / SageMaker / QuickSight  ← consumo
```

> S3 Tables + SageMaker Lakehouse é a aposta da AWS para substituir arquiteturas onde você tinha separadamente um Data Lake (S3 + Glue) e um Data Warehouse (Redshift) — convergindo tudo em uma camada Iceberg unificada.

## Hands On

1. Criar S3 Table Bucket

![[Pasted image 20260410152107.png]]

![[Pasted image 20260410152129.png]]

![[Pasted image 20260410152139.png]]

2. Dentro da Table Bucket criada, criar a Namespace. Depois criar

| Athena UI                       | Conceito real    |
| ------------------------------- | ---------------- |
| Data source (AwsDataCatalog)    | Glue Catalog     |
| Catalog (`s3tablescatalog/...`) | **Table Bucket** |
| Database (`us_data`)            | **Namespace**    |
| Table (`state_population`)      | Table            |
AwsDataCatalog -> Ele está ali porque o Athena precisa de um **catálogo de metadados** para saber quais tabelas existem.
 └── s3tablescatalog/gustavo-brandao...   ← Table Bucket
       └── us_data                        ← Namespace
             └── state_population        ← Table
    