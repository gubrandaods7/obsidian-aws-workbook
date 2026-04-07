ETL significa Extração, Transformação e Carregamento. É o processo usado para mover dados de fontes para um Data Warehouse.

## Extract (E)

- Buscar dados crus de diferentes fontes/sistemas. Podendo ser de banco de dados, CRMs, arquivos crus, APIs ou qualquer outro repositório de dados
- Garante a integridade dos dados durante a fase de extração
- Pode ser feito em tempo real ou em batch, depende dos requisitos

## Transform (T)

- Converte os dados extraídos para um formato que se adeque para o Data Warehouse escolhido
- Pode envolver diversas operações diferentes:
	- Data cleansing (remover duplicadas, ajustar erros)
	- Data enrichment (complementar com dados de outras fontes)
	- Format changes (formatação de data, manipulação de strings)
	- Aggregations or computations (cálculos de totais e médias)
	- Encoding or decoding
	- Handling missing values

## Load (L)

- Move os dados transformados para o Data Warehouse escolhido, ou algum outro repositório de dados
- Pode ser feito em batches ou via transmissão (quando o dado fica disponível)
- Se assegura que os dados se mantém íntegros nesse processo de carregamento

---

## ETL vs ELT

Uma distinção importante em Data Engineering:

| | ETL | ELT |
|---|---|---|
| **Ordem** | Transforma antes de carregar | Carrega primeiro, transforma depois |
| **Onde transforma** | Em um servidor/engine externo | Dentro do próprio destino (DW/Data Lake) |
| **Ideal para** | Data Warehouses tradicionais | Data Lakes e DWs modernos na nuvem |
| **Exemplos AWS** | AWS Glue | Amazon Redshift + dbt, Athena |

> No contexto AWS, o padrão ELT é muito comum: dados crus são carregados no S3 (Data Lake) e transformados posteriormente com Glue, Athena ou Redshift Spectrum.

## Staging Area

Área intermediária onde os dados extraídos são temporariamente armazenados antes da transformação. Serve para:
- Isolar a fonte de dados do processo de transformação
- Permitir reprocessamento sem precisar re-extrair
- Facilitar auditoria e debug de problemas na pipeline

Na AWS, o S3 frequentemente atua como staging area.

## Gerenciamento de Pipelines ETL

Esse processo deve ser automatizado de alguma forma:

- **AWS Glue** — serviço gerenciado de ETL/ELT

Sistemas de orquestração:
- **Amazon EventBridge** — trigger baseado em eventos
- **Amazon Managed Workflows for Apache Airflow (MWAA)** — orquestração complexa
- **AWS Step Functions** — orquestração de workflows serverless
- **AWS Lambda** — execução de funções trigger-based
- **Glue Workflows** — orquestração nativa dentro do Glue

## Monitoramento de Pipelines

Uma pipeline ETL em produção precisa ser monitorada:
- **Falhas de extração** — fonte indisponível, schema mudou
- **Qualidade dos dados** — valores nulos inesperados, duplicatas
- **Latência** — pipeline demorando mais que o esperado

Na AWS: **Amazon CloudWatch** para métricas e alertas, **AWS Glue Data Quality** para validação dos dados.