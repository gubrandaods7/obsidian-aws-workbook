
Data Modeling é o processo de definir como os dados são estruturados, organizados e relacionados entre si em um sistema de armazenamento.

---

## Star Schema

O Star Schema (Esquema Estrela) é o modelo dimensional mais comum em Data Warehouses. Seu nome vem do formato visual: uma tabela central rodeada por tabelas de dimensão.

### Tabela Fato
- Armazena os **eventos ou transações mensuráveis** do negócio
- Contém métricas numéricas (valores, quantidades, durações)
- Contém chaves estrangeiras que apontam para as dimensões
- Exemplos: fato_vendas, fato_reservas, fato_cliques

### Tabelas de Dimensão
- Descrevem o **contexto** dos eventos da tabela fato
- Contêm atributos descritivos (nomes, categorias, datas)
- Exemplos: dim_cliente, dim_produto, dim_tempo, dim_localidade

### Chaves
- **Chave Primária (PK):** identifica unicamente cada registro em uma tabela
- **Chave Estrangeira (FK):** referencia a PK de outra tabela, criando o relacionamento

### Vantagens
- Simples de entender e consultar
- Bom desempenho em queries analíticas
- Fácil integração com ferramentas de BI

ERD (Entity Relationship Diagram)
![[Pasted image 20260407110547.png]]

---

## Snowflake Schema

Variação do Star Schema onde as tabelas de dimensão são **normalizadas** — ou seja, subdivididas em tabelas menores.

- **Vantagem:** menos redundância de dados
- **Desvantagem:** queries mais complexas com mais JOINs
- Usado quando o espaço de armazenamento é prioridade ou quando as dimensões são muito grandes

**Exemplo:** em vez de uma única `dim_produto` com categoria e subcategoria, o Snowflake teria `dim_produto` → `dim_categoria` → `dim_subcategoria`.

---

## Star vs Snowflake

| | Star Schema | Snowflake Schema |
|---|---|---|
| **Normalização** | Desnormalizado | Normalizado |
| **Complexidade de queries** | Baixa (menos JOINs) | Alta (mais JOINs) |
| **Performance** | Mais rápido para leitura | Mais lento para leitura |
| **Redundância de dados** | Maior | Menor |
| **Uso comum** | BI e dashboards | DWs com dimensões complexas |

---

## Data Lineage

Representação visual que mostra o fluxo e a transformação do dado durante seu ciclo de vida. Da fonte até sua destinação final.

- Responde perguntas como: *"De onde veio esse dado?"*, *"Quem o transformou?"*, *"Onde ele é usado?"*
- Fundamental para **auditoria**, **debugging de pipelines** e **conformidade regulatória**
- Na AWS: **AWS Glue** registra lineage de jobs ETL; **Amazon SageMaker** tem lineage tracking para ML

![[Pasted image 20260407110949.png|665]]

![[Pasted image 20260407111023.png|664]]
