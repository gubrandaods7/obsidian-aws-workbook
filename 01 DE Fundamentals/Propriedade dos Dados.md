
# Os Vs do Big Data

As propriedades dos dados em contextos de Big Data são descritas pelos **"Vs"** — características que definem o desafio de trabalhar com grandes volumes de dados. Os 3 Vs originais foram propostos por Doug Laney em 2001, e ao longo do tempo outros foram adicionados.

---

## 1. Volume

Se refere à quantidade ou tamanho dos dados que as organizações lidam em um período de tempo.

### Características
- Podem variar de gigabytes até petabytes e além
- Desafios em armazenamento, processamento e análise de grandes volumes
- Exige soluções distribuídas (não cabe em uma única máquina)

### Exemplos
- Redes sociais processando terabytes de dados diariamente (texto, fotos, vídeos)
- Grandes varejistas acumulando petabytes de transações ao longo dos anos

### Na AWS
- **Amazon S3** — armazenamento escalável e de baixo custo para grandes volumes
- **Amazon Redshift** — DW para análise de grandes volumes estruturados

---

## 2. Velocidade

Se refere à velocidade na qual novos dados são gerados, coletados e processados.

### Características
- Velocidades altas requerem processamento em tempo real ou quase real
- Define a escolha entre **Batch** (processa acumulado) e **Streaming** (processa evento a evento)
- Ingestão rápida pode ser crítica dependendo da aplicação

### Exemplos
- Sensores de IoT transmitindo dados a cada milissegundo
- Sistemas de high-frequency trading onde cada milissegundo faz diferença

### Na AWS
- **Amazon Kinesis** — ingestão e processamento de dados em streaming
- **AWS Lambda** — processamento serverless em tempo real

---

## 3. Variedade

Se refere aos diferentes tipos e formatos dos dados e suas fontes.

### Características
- Dados podem ser estruturados, não-estruturados ou semi-estruturados
- Vêm de múltiplas fontes em formatos diferentes
- Exige pipelines capazes de lidar com heterogeneidade

### Exemplos
- Negócio que analisa tabelas relacionais (estruturado), emails (não-estruturado) e logs JSON (semi-estruturado)
- Sistema de saúde com registros médicos eletrônicos, dispositivos vestíveis e formulários de feedback

### Na AWS
- **AWS Glue** — cataloga e processa dados de diferentes formatos e fontes
- **Amazon Athena** — consulta dados em múltiplos formatos diretamente no S3

---

## 4. Veracidade

Se refere à **confiabilidade e qualidade** dos dados.

### Características
- Dados podem ser incompletos, inconsistentes ou incorretos
- Ruído e incerteza nos dados afetam a qualidade das análises
- Exige processos de validação, limpeza e governança

### Exemplos
- Dados de sensores com falhas de leitura
- Registros duplicados ou com campos nulos

### Na AWS
- **AWS Glue Data Quality** — validação e monitoramento da qualidade dos dados

---

## 5. Valor

O V mais importante — se refere ao **valor de negócio** que pode ser extraído dos dados.

### Características
- Ter grande volume de dados não significa nada se não gera insight
- O objetivo final de qualquer pipeline de dados é gerar valor
- Justifica o investimento em infraestrutura e engenharia de dados

---

## Resumo

| V | Pergunta | Desafio |
|---|----------|---------|
| **Volume** | Quanto dado? | Armazenar e processar em escala |
| **Velocidade** | Quão rápido chega? | Processar em tempo real |
| **Variedade** | Em que formato? | Integrar fontes heterogêneas |
| **Veracidade** | É confiável? | Garantir qualidade e consistência |
| **Valor** | Vale a pena? | Extrair insights acionáveis |