
Amazon S3 (Simple Storage Service) é o serviço de armazenamento de objetos da AWS e um dos principais blocos de construção da plataforma. Funciona como a "espinha dorsal" de inúmeros serviços AWS e é a base de praticamente qualquer arquitetura de Data Lake na nuvem.

## Índice

- [[#Casos de Uso]]
- [[#Buckets]]
- [[#Objects (Objetos)]]
- [[#Segurança]]
- [[#Versionamento]]
- [[#Storage Classes]]
- [[#Lifecycle Policies]]
- [[#Replicação]]
- [[#Performance]]
- [[#Event Notifications]]
- [[#S3 como Data Lake]]
- [[#Integração com Serviços AWS]]
- [[#Preços (conceitos)]]
- [[#Boas Práticas para Data Engineering]]

---

## Casos de Uso

- Backup e armazenamento de dados
- Recuperação de desastres (Disaster Recovery)
- Arquivamento de longo prazo
- Armazenamento em nuvem híbrida
- Hospedagem de aplicações e artefatos
- Media hosting (imagens, vídeos, áudios)
- **Data Lakes e Big Data Analytics**
- Hospedagem de sites estáticos
- Distribuição de conteúdo estático via CloudFront

---

## Buckets

- Objetos (arquivos) são armazenados dentro de **buckets** (análogos a diretórios raiz)
- O nome do bucket deve ser **globalmente único** em todas as regiões e contas AWS
- Buckets são criados em uma **região específica**, mesmo que o S3 seja um serviço global
- Convenções de nomenclatura: apenas letras minúsculas, números e hífens; sem underscores; sem endereços IP; deve começar com letra ou número

---

## Objects (Objetos)

- Todo objeto possui uma **chave (key)**, que é o caminho completo dentro do bucket
  - Exemplo: `s3://my-bucket/my_file.txt`
  - Exemplo: `s3://my-bucket/my_folder/subfolder/my_file.txt`
- A chave = **prefixo** + **nome do objeto**
  - Prefixo: `my_folder/subfolder/`
  - Objeto: `my_file.txt`
- **Não existe conceito real de diretórios** — o que existe são chaves com `/` no nome
- Isso tem implicações importantes para particionamento em Data Lakes

![[Pasted image 20260408085026.png|151]]

### Limites e Metadados

| Propriedade | Detalhe |
|---|---|
| Tamanho máximo do objeto | 5 TB |
| Multipart Upload obrigatório acima de | 5 GB |
| Recomendado usar Multipart acima de | 100 MB |
| Metadados do sistema/usuário | Pares chave-valor (texto) |
| Tags | Até 10 pares chave-valor Unicode |
| Version ID | Disponível quando versionamento está habilitado |

---

## Segurança

### Controle de Acesso

**Baseado em recurso (Resource-based):**
- **Bucket Policies** — regras JSON aplicadas ao bucket; permitem acesso cross-account
- **Object ACL (Access Control List)** — controle granular por objeto (pode ser desabilitado)
- **Bucket ACL** — menos comum; também pode ser desabilitado

**Baseado em identidade (IAM-based):**
- Policies IAM atachadas a usuários/roles/grupos

> Um usuário IAM pode acessar um objeto S3 se:
> - A IAM policy permitir **OU** a Bucket Policy permitir
> - E não existir um DENY explícito

### Block Public Access

- Configuração adicional para garantir que o bucket **nunca seja público**, independentemente das policies
- Deve ser ativado em buckets que não precisam de acesso público (padrão recomendado)
- Pode ser configurado no nível da conta AWS inteira

### Encryption

| Tipo | Descrição |
|---|---|
| SSE-S3 | Chaves gerenciadas pela AWS (padrão desde 2023) |
| SSE-KMS | Chaves gerenciadas pelo AWS KMS; trilha de auditoria via CloudTrail |
| SSE-C | Chaves fornecidas pelo cliente; AWS não armazena a chave |
| Client-side | Criptografia feita antes do upload |

> Para Data Engineering: SSE-KMS é comum em ambientes regulados pois permite auditoria de quem acessou as chaves.

---

## Versionamento

- Habilitado no nível do bucket
- Mantém múltiplas versões do mesmo objeto
- Protege contra deleção acidental e sobrescrita
- Ao deletar um objeto versionado, a AWS insere um **Delete Marker** (não apaga fisicamente)
- Para apagar de verdade, é necessário deletar a versão específica
- Objetos enviados antes do versionamento ser habilitado têm version ID `null`

---

## Storage Classes

Escolher a classe correta é fundamental para otimização de custo em Data Lakes.

| Classe | Uso | Disponibilidade | Custo relativo |
|---|---|---|---|
| **S3 Standard** | Dados acessados frequentemente | 99.99% | Alto |
| **S3 Intelligent-Tiering** | Padrão de acesso desconhecido ou variável | 99.9% | Médio (+ taxa de monitoramento) |
| **S3 Standard-IA** | Acesso infrequente, recuperação rápida | 99.9% | Médio-baixo |
| **S3 One Zone-IA** | Infrequente, pode ser recriado, 1 AZ | 99.5% | Baixo |
| **S3 Glacier Instant Retrieval** | Arquivamento com acesso em ms | 99.9% | Muito baixo |
| **S3 Glacier Flexible Retrieval** | Arquivamento, acesso em minutos a horas | 99.99% | Muito baixo |
| **S3 Glacier Deep Archive** | Arquivamento de longo prazo, acesso em horas | 99.99% | Mínimo |

> Regra prática para Data Lakes:
> - Zona **Raw/Landing** → Standard ou Intelligent-Tiering
> - Zona **Curated/Trusted** com dados antigos → Standard-IA
> - Dados de **compliance/auditoria** raramente acessados → Glacier

---

## Lifecycle Policies

Regras automáticas para transição entre storage classes ou expiração de objetos.

Exemplo de uso:
```
Após 30 dias   → mover para Standard-IA
Após 90 dias   → mover para Glacier Instant Retrieval
Após 365 dias  → expirar (deletar)
```

- Podem ser aplicadas a prefixos específicos (partições do Data Lake)
- Podem filtrar por tags de objetos
- Suportam gerenciamento de versões antigas

---

## Replicação

| Tipo | Descrição |
|---|---|
| **CRR** (Cross-Region Replication) | Replica entre regiões diferentes |
| **SRR** (Same-Region Replication) | Replica dentro da mesma região |

- Requer versionamento habilitado em ambos os buckets (origem e destino)
- Replicação é **assíncrona**
- Objetos existentes antes da replicação ser configurada **não são replicados automaticamente** (usar S3 Batch Replication)
- Delete markers podem ou não ser replicados (configurável)

**Casos de uso:**
- CRR: compliance, baixa latência em outra região, replicação cross-account
- SRR: agregação de logs, replicação entre contas de prod e teste

---

## Performance

### Prefixos e Paralelismo

- S3 suporta **3.500 PUT/COPY/POST/DELETE** e **5.500 GET/HEAD** por segundo **por prefixo**
- Distribua objetos em múltiplos prefixos para maximizar throughput
- Importante para jobs de leitura massiva no Spark/EMR

### Upload Otimizado

| Técnica | Quando usar |
|---|---|
| **Multipart Upload** | Arquivos > 100 MB; obrigatório > 5 GB |
| **S3 Transfer Acceleration** | Uploads de longa distância via edge locations do CloudFront |
| **Byte-range fetches** | Leitura paralela de partes específicas; útil para Parquet |

### S3 Select e Glacier Select

- Permite filtrar dados **dentro do S3** usando SQL simples
- Reduz transferência de dados e custo
- Suporta CSV, JSON, Parquet (comprimido com GZIP/BZIP2)
- Alternativa leve ao Athena para queries simples

---

## Event Notifications

S3 pode emitir eventos para:
- **SNS** (notificações)
- **SQS** (filas de processamento)
- **Lambda** (processamento serverless)
- **EventBridge** (roteamento avançado para múltiplos destinos)

Eventos disponíveis: `s3:ObjectCreated`, `s3:ObjectRemoved`, `s3:ObjectRestore`, `s3:Replication`, entre outros.

> Caso de uso em Data Engineering: trigger de pipeline ao chegar novo arquivo na zona Raw do Data Lake.

---

## S3 como Data Lake

### Estrutura de Zonas Típica

```
s3://my-datalake/
├── raw/              ← dados brutos, imutáveis, particionados por data
│   └── source=crm/year=2025/month=04/day=08/
├── trusted/          ← dados limpos, validados, formato Parquet
│   └── domain=sales/year=2025/month=04/
└── refined/          ← agregações e modelos prontos para consumo
    └── subject=revenue/
```

### Particionamento

- Particionamento por `year/month/day` é padrão para dados temporais
- Ferramentas como Athena, Glue, Spark reconhecem partições Hive-style automaticamente
- Evite partições com **cardinalidade muito alta** (ex: por UUID) — gera overhead de metadados

### Formatos de Arquivo

| Formato | Vantagem | Quando usar |
|---|---|---|
| **Parquet** | Columnar, comprimido, eficiente para analytics | Padrão para camadas Trusted/Refined |
| **ORC** | Similar ao Parquet, melhor integração com Hive/EMR | Workloads Hive-heavy |
| **Avro** | Row-based, ótimo para schema evolution | Streaming / Kafka |
| **JSON** | Flexível, legível | Raw layer, APIs |
| **CSV** | Universalmente suportado | Ingestão inicial, legado |

---

## Integração com Serviços AWS

| Serviço | Como usa o S3 |
|---|---|
| **Athena** | Consulta diretamente arquivos no S3 via SQL |
| **Glue** | ETL; usa S3 como fonte e destino; Glue Catalog aponta para S3 |
| **EMR** | Processa dados do S3 com Spark/Hive |
| **Redshift Spectrum** | Consulta dados externos no S3 sem carregar no cluster |
| **Lake Formation** | Governa permissões de Data Lakes no S3 |
| **Kinesis Firehose** | Entrega streams diretamente para S3 |
| **SageMaker** | Usa S3 para datasets de treino e artefatos de modelos |

---

## Preços (conceitos)

- Cobrado por: **armazenamento (GB/mês)**, **requisições** (GET, PUT, etc.), **transferência de dados saindo da AWS**
- Transferência entre S3 e serviços na mesma região geralmente é gratuita
- S3 para internet: cobrado por GB transferido
- Usar **VPC Endpoint para S3** evita tráfego pela internet e pode reduzir custo e latência

---

## Boas Práticas para Data Engineering

1. Sempre usar **Parquet + compressão Snappy** para dados analíticos
2. Definir **Lifecycle Policies** para dados históricos
3. Usar **particionamento Hive-style** para compatibilidade com Athena/Glue
4. Habilitar **versionamento** em buckets de dados críticos
5. Bloquear acesso público com **Block Public Access**
6. Usar **SSE-KMS** em dados sensíveis para auditoria de acesso
7. Configurar **Event Notifications** para acionar pipelines automaticamente
8. Separar buckets por ambiente (dev, staging, prod)
9. Usar **S3 Inventory** para auditoria de objetos em larga escala
10. Monitorar custos com **S3 Storage Lens**
