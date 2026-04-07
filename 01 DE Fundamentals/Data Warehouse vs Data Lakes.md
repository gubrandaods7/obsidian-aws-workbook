
# Data Warehouse

Repositório centralizado otimizado para análises onde dados que diferentes fontes são armazenados em formato estruturado.

## Características
- São desenhados para análises e queries complexas
- Os dados são limpos, transformados e carregados (ETL)
- Tipicamente usam os esquemas [[Star]] ou [[Snowflake]]
- Otimizados para operações pesadas de leitura

## Exemplo
- Amazon Redshift
- Google Big Query

![[Pasted image 20260407095743.png]]
- Criação de Views baseadas em necessidades internas
# Data Lake

É um repositório de armazenamento que guarda uma vasta quantidade de dados crus em seu formato nativo, incluindo dados estruturados, não estruturados e semi-estruturados.

## Características

- Pode armazenar quantidades enormes de dados crus sem um esquema pré-definido
- Os dados são carregados como eles vêm, sem necessidade de pré-processamento
- Suporta carregamento em batch, tempo-real e em transmissão
- Pode ser consultado para transformação ou exploração dos dados

## Exemplo
-[[ Amazon Simple Storage Service (Amazon S3) ]]-> [[AWS Glue]] -> [[Amazon Athena]]

## Comparações

| Característica     | Data Warehouse                                         | Data Lake                                                           |
| ------------------ | ------------------------------------------------------ | ------------------------------------------------------------------- |
| **Schema**         | Schema-on-write (schema predefinido antes de escrever) | Schema-on-read (schema definido no momento da leitura)              |
| **Processamento**  | ETL (Extract, Transform, Load)                         | ELT (Extract, Load, Transform) ou apenas Load                       |
| **Tipos de Dados** | Principalmente dados estruturados                      | Estruturados, semi-estruturados e não-estruturados                  |
| **Agilidade**      | Menos ágil — schema rígido e predefinido               | Mais ágil — aceita dados crus sem estrutura prévia                  |
| **Custo**          | Mais caro — otimizado para queries complexas           | Armazenamento barato, mas custos sobem ao processar grandes volumes |

###  Escolha 

#### Escolher Data Warehouse quando
- Os dados que possuímos são estruturados e requerem consultas rápidas e complexas
- Integração dos dados de diferentes fontes é essencial
- BI e Analytics são os principais recursos que buscamos

#### Escolher Data Lake Warehouse quando
- Os dados que possuímos são um mix de estruturados, não estruturados e semi-estruturados
- É necessário uma estrutura escalável e de baixo custo para guardar grandes volumes de dados
- Não existe definição exata do que sera feito com os dados, e você precisa de flexibilidade no armazenamento e processamento
- Analytics avançado, M.L e data discovery são recursos importantes

#### Ambos
- É possível a combinação de ambos, onde os dados crus são ingeridos e posteriormente após serem processados e refinados são enviados para um DW para análises


# Data Lakehouse

É uma arquitetura híbrida que combina o melhor dos dois mundos.
Procuram trazer a confiabilidade, performance e capacidades de um DW, enquanto mantém a flexibilidade, escala e o custo baixo de armazenamento de um DL.

## Características

- Suporta todos os tipos de dados
- Permite esquemas na leitura e em armazenamento
- Fornece capacidades tanto para análises detalhadas quanto para atividades de M.L
- Tipicamente construídas em cima de uma Cloud ou arquiteturas distribuídas
- Se beneficia das tecnologias de DL, que traz transações [[ACID]] para big data

## Exemplos

- AWS Lake Formation (com S3 e Redshift Spectrum)
- Delta lake
- Databricks Lakehouse Platform
- Azure Synapse Analytics