
# Data Mesh

Não se trata de uma tecnologia, mas sim uma **abordagem de governança e organização dos dados** — quem acessa, quem cuida, como é acessada.

---

## Conceito

Data Mesh é uma arquitetura descentralizada onde a **responsabilidade pelos dados é distribuída entre os times de domínio**, ao invés de centralizada em uma única equipe de dados.

Equipes individuais são donas dos produtos de dados de seu domínio. Esses produtos são disponibilizados e consumidos por outras pessoas na mesma organização sem que precisem construir pipelines do zero.

**Exemplo:** Se alguém de marketing precisa de dados de transações, não precisa construir uma pipeline completa de extração e tratamento — ela consome o produto de dados que o time financeiro já gerencia e disponibiliza.

![[Pasted image 20260407102103.png]]

---

## Os 4 Princípios do Data Mesh

### 1. Domain Ownership (Propriedade por Domínio)
- Cada domínio de negócio (vendas, marketing, financeiro) é responsável pelos seus próprios dados
- O time que gera o dado é o mesmo que o mantém e disponibiliza
- Elimina o gargalo de uma equipe central de dados sendo responsável por tudo

### 2. Data as a Product (Dado como Produto)
- Os dados devem ser tratados como produtos: com qualidade, documentação, SLA e fácil consumo
- Cada produto de dado tem um "dono" responsável pela sua qualidade e disponibilidade
- Características de um bom produto de dado: descobrível, confiável, seguro, interoperável

### 3. Self-serve Data Infrastructure (Infraestrutura Self-serve)
- A plataforma central fornece ferramentas e infraestrutura para que os times de domínio possam publicar e consumir dados de forma autônoma
- Os times não precisam depender de um time central para criar pipelines
- Na AWS: serviços como **AWS Glue**, **Amazon Athena**, **Lake Formation** compõem essa infraestrutura

### 4. Federated Computational Governance (Governança Federada)
- Regras globais de governança (segurança, privacidade, compliance) são definidas centralmente
- A implementação dessas regras é responsabilidade de cada domínio
- Equilíbrio entre autonomia dos times e conformidade organizacional

---

## Quando usar Data Mesh

| Situação | Indicação |
|----------|-----------|
| Organização grande com muitos domínios de negócio | Sim |
| Times de dados sobrecarregados e sendo gargalo | Sim |
| Empresa pequena com poucos times | Não — overhead desnecessário |
| Necessidade de governança centralizada rígida | Com cautela |

---

## Data Mesh vs Arquitetura Centralizada

| | Centralizada | Data Mesh |
|---|---|---|
| **Responsabilidade** | Time central de dados | Times de domínio |
| **Escalabilidade** | Limitada pelo time central | Alta — cresce com os domínios |
| **Qualidade** | Centralizada | Distribuída por domínio |
| **Agilidade** | Menor (depende de um time) | Maior (times autônomos) |
