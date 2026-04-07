
## 1. Visão Geral

Existem três tipos de dados.

## 2. Tipos
### Estruturados
#### Definição

Dados estruturados são dados com um formato padronizado. Ele é normalmente tabular, com linhas e colunas, que definem de forma clara seus atributos dos dados.

Os dados estruturados tem seus atributos definidos, ou seja, um exemplo de registro de reserva de hotéis. Todo registro da reserva pode contar o nome, nome do evento, data, valor, datas de entrada e saída, etc. E isso vai se repetir para cada novo registro.

Eles são relacionais, ou seja, tem valores comuns que podem se integrar com outros conjuntos diferentes de dados. Seguindo o mesmo exemplo dos dados de reservas de hotéis, cada registro tende a possuir um cliente, aquele que faz a reserva. 

Nesse caso, podemos relacionar esses dados do cliente, como por exemplo o nome, e-mail, endereço, telefone, etc com os dados das reservas. Isso pode ser feito por exemplo um id único "customer_id" para cada cliente, e para a reserva, podemos atribuir um "bookin_id".

Os dados estruturados são quantitativos, ou seja, são mensuráveis, podem ser usados para operações matemáticas. Com uma estrutura padronizada de dados, podemos simplesmente contar, somar, medir, realizar cálculos de atributos com os valores. 

Podemos por exemplo facilmente contar a quantidade de bookings/reservas em determinado período, podemos calcular da mesma forma esse mesmo valor considerando apenas meses, dias da semana ou anos em específico, etc.

Alguns exemplos mais comuns de dados estruturados:
- Arquivos Excel (.xlsx)
- Arquivos CSV (.csv)
- Banco de Dados SQL / Tabela de bancos de dados
- Dados de pontos de vendas
- Controle de inventário
- Sistemas de reserva

### 2. Não-Estruturados

Dados não estruturados por sua vez são aqueles que não tem um esquema de dados ou estrutura definidos.

Não são armazenados em base de dados convencionais, mas são usualmente armazenados em base de dados não-relacionais (NoSQL), ou em Data Lakes.

Por não serem estruturados, eles apresentam uma dificuldade maior para serem análisados, pois não seguem um padrão. Para isso são necessários formas mais sofisticadas de análise, como por exemplo Machine Learning, NLP, para que dessa forma seja possível extrair insights e informações relevantes desses dados.

A grande maioria dados não-estruturados seguem o formato de textos, como por exemplo: e-mails, documentos em Word ou PDFs, posts de blogs e de redes sociais.
E também a parte não-textual, como arquivos de imagens: GIF, PNG, JPG), videos, dados de atividade de celulares, dados de sensores, etc.


### 3. Semi-Estruturados

São a combinação entre os dois principais mundos. São dados que não seguem um esquema rígido de tabela relacional, mas também não são totalmente desorganizados.

Possuem uma certa estrutura que é flexível.

É muito como encontrarmos esse tipo de dados em arquivos JSON, XML, Parquet, etc.

Seguindo o exemplo do JSON, ele é considerado semi-estruturado pois:
- Possui estrutura
- Mas seus campos podem mudar, não são fixos

Exemplo simples:
```JSON
{
  "id": 1,
  "nome": "Gustavo",
  "idade": 30
}
```

Agora um outro exemplo:
```JSON
{
  "id": 2,
  "nome": "Ana",
  "endereco": {
    "cidade": "São Paulo",
    "cep": "00000-000"
  },
  "interesses": ["dados", "python"]
}
```

Com isso conseguimos observar que os campos podem mudar, os valores podem variar. A estrutura pode ser diferente por cada registro. E isso define um dado semi-estruturado.
## 3. Quando usar
- Structured → analytics, BI
- Semi → APIs, logs
- Unstructured → raw ingestion


## 4. Conceitos Relacionados
- [[data-lake]]
- [[schema-evolution]]

