
## JDBC
- **Java Database Connectivity**
- Independente de plataforma (roda em qualquer SO)
- Dependente de linguagem (Java/JVM)
- Usado por ferramentas como AWS Glue para conectar a bancos relacionais (MySQL, PostgreSQL, Redshift)

## ODBC
- **Open Database Connectivity**
- Dependente de plataforma (drivers específicos por SO)
- Independente de linguagem (pode ser usado por Python, R, etc.)
- Padrão mais antigo, comum em ambientes Windows e ferramentas de BI como Tableau e Power BI

## Raw Logs

Arquivos de log gerados por sistemas, aplicações e infraestrutura. Geralmente não estruturados ou semi-estruturados.

- **Exemplos:** logs de servidores web (Apache, Nginx), logs de aplicação, logs de acesso AWS (CloudTrail, S3 Access Logs)
- **Formato comum:** texto linha a linha, JSON Lines
- **Na AWS:** logs são frequentemente enviados ao S3 ou CloudWatch Logs para posterior processamento

## APIs

Interface que permite sistemas distintos se comunicarem e trocarem dados.

- **REST APIs** — padrão mais comum, retorna JSON ou XML via HTTP
- **GraphQL** — permite consultas flexíveis, retorna exatamente os campos solicitados
- **Webhook** — API reversa: o sistema envia dados automaticamente quando um evento ocorre (push, não pull)
- **Na AWS:** Amazon API Gateway expõe APIs; AWS Lambda pode consumir ou servir APIs

## Streams of data

Fluxo contínuo de dados gerados em tempo real, processados à medida que chegam.

- **Diferença do Batch:** batch processa dados acumulados em intervalos; streaming processa evento a evento
- **Casos de uso:** monitoramento em tempo real, detecção de fraudes, telemetria de IoT
- **Na AWS:** **Amazon Kinesis Data Streams** e **Amazon Kinesis Firehose** são os principais serviços para ingestão de streams

---

# Data Formats

## CSV
- Formato tabular simples, separado por vírgulas
- Legível por humanos, amplamente suportado
- **Limitação:** sem tipagem, sem compressão nativa, ineficiente para grandes volumes

## JSON
- Formato semi-estruturado baseado em chave-valor
- Muito usado em APIs e logs
- Suporta estruturas aninhadas (objetos dentro de objetos)
- **Limitação:** verboso, sem schema fixo pode gerar inconsistências

## Avro
- Formato binário com schema embutido (definido em JSON)
- Ideal para serialização de dados em streaming
- **Vantagem:** schema evolution — permite adicionar/remover campos sem quebrar consumidores
- Muito usado com **Apache Kafka**

## Parquet
- Formato colunar (armazena dados por coluna, não por linha)
- Excelente compressão e performance para queries analíticas
- Ideal para leitura de colunas específicas sem ler o arquivo inteiro
- **Na AWS:** formato preferido para Data Lakes no S3, usado nativamente por Athena, Glue e Redshift Spectrum

### Comparativo de Formatos

| Formato | Estrutura | Schema | Melhor para |
|---------|-----------|--------|-------------|
| CSV | Linha | Nenhum | Dados simples, interoperabilidade |
| JSON | Linha | Flexível | APIs, logs, semi-estruturado |
| Avro | Linha (binário) | Embutido | Streaming, Kafka |
| Parquet | Colunar | Embutido | Analytics, Data Lake, queries OLAP |

![[Pasted image 20260407105844.png|304]]