
## Índice

- [[#Pré-requisitos]]
- [[#Casos de Uso]]
- [[#Comportamentos Importantes]]
- [[#Filtros de Replicação]]
- [[#Storage Class no Destino]]
- [[#Replication Time Control (RTC)]]
- [[#Segurança e Criptografia]]
- [[#Replicação em Data Lakes]]
- [[#Hands-on - SRR]]

---

| Tipo | Nome completo | Escopo |
|---|---|---|
| **CRR** | Cross-Region Replication | Buckets em regiões diferentes |
| **SRR** | Same-Region Replication | Buckets na mesma região |

---

## Pré-requisitos

- **Versionamento habilitado** em ambos os buckets (origem e destino)
- **IAM Role** com permissões para que o S3 leia da origem e escreva no destino
- Os buckets podem estar em **contas AWS diferentes**
- A cópia é **assíncrona** — acontece em background, com latência de segundos a minutos

![[Pasted image 20260408104944.png]]

---

## Casos de Uso

### CRR — Cross-Region Replication
- **Compliance e soberania de dados**: regulações que exigem cópias em geografias diferentes
- **Redução de latência**: servir dados de uma região mais próxima do usuário final
- **Replicação cross-account**: manter cópia em uma conta separada (ex: conta de DR)

### SRR — Same-Region Replication
- **Agregação de logs**: consolidar logs de múltiplos buckets em um único bucket central
- **Ambientes separados**: sincronizar dados de produção para um bucket de testes/homologação na mesma região
- **Conformidade interna**: manter cópia em uma conta diferente dentro da mesma região

---

## Comportamentos Importantes

### Objetos existentes
- Após habilitar a replicação, **apenas novos objetos são replicados automaticamente**
- Para replicar objetos já existentes, use **S3 Batch Replication**
  - Também útil para replicar objetos que falharam anteriormente

### Deleção
- **Delete Markers** podem ser replicados — mas é **opcional e configurável** separadamente
- **Deleções com Version ID específico não são replicadas** — isso previne deleções maliciosas ou acidentais de se propagarem para o destino

### Chaining (encadeamento)
- Se o Bucket A replica para o Bucket B, e o Bucket B replica para o Bucket C:
  - Objetos criados no **Bucket A não chegam ao Bucket C** automaticamente
  - Replicação **não é transitiva** — cada par origem → destino é independente

---

## Filtros de Replicação

É possível replicar **apenas parte** do bucket, filtrando por:
- **Prefixo** — ex: replicar apenas `raw/source=crm/`
- **Tags** — ex: replicar apenas objetos com tag `env=prod`
- Combinação de prefixo + tags

Isso permite, por exemplo, replicar apenas a camada Raw de um Data Lake para outra região, sem replicar dados intermediários.

---

## Storage Class no Destino

Por padrão, os objetos replicados mantêm a mesma Storage Class da origem.
É possível configurar uma classe diferente no destino — útil para reduzir custo em buckets de DR que raramente são acessados (ex: destino em Glacier).

---

## Replication Time Control (RTC)

- Feature opcional que garante que **99,99% dos objetos sejam replicados em até 15 minutos**
- Inclui métricas e notificações de progresso via CloudWatch
- Tem custo adicional
- Recomendada para workloads que exigem RPO (Recovery Point Objective) rigoroso

---

## Segurança e Criptografia

- Objetos criptografados com **SSE-S3** são replicados sem configuração adicional
- Objetos com **SSE-KMS** requerem configuração explícita da chave KMS de destino e permissões adicionais na IAM Role
- Objetos com **SSE-C** (chave do cliente) **não podem ser replicados**

---

## Replicação em Data Lakes

Padrões comuns:

| Cenário | Tipo | Motivação |
|---|---|---|
| Backup de dados regulatórios | CRR | DR em outra região + compliance geográfico |
| Dev/Test com dados reais | SRR | Espelhar prod para teste sem expor o bucket original |
| Consolidação de logs multi-conta | SRR | Centralizar em conta de segurança/observabilidade |
| Latência global em leitura | CRR | Replicar Refined layer para região dos consumidores |

## Hands-on - SRR
https://www.udemy.com/course/aws-data-engineer/learn/lecture/40285946#overview

### 1. Criar Bucket Origem e Destino

No meu caso, o origem é o `gustavo-brandao-replication-s3-v1-027087671762-us-east-2-an`. E o de destino `gustavo-brandao-replication-s3-v2-027087671762-us-east-1-an`.



![[Pasted image 20260408132514.png]]

### 2. Ativar versionamento em ambos os buckets

![[Pasted image 20260408132614.png]]

### 3. Criar replication Rule no Bucket de Origem

Entrar em `Management` > `Create replication rule`

![[Pasted image 20260408132700.png]]

#### 3.1 Atribuir um nome

![[Pasted image 20260408132745.png]]

#### 3.2 Selecionar opção para aplicar pra todos os objetos do Bucket

![[Pasted image 20260408132820.png]]

#### 3.3 Escolher o Bucket de Destino (Onde sera replicado)

![[Pasted image 20260408132840.png]]

#### 3.4 Criar uma nova role

Nesse caso, eu já havia criado antes da print

![[Pasted image 20260408132857.png]]

#### 3.5 Salvar

Após salvar, ele vai perguntar se deseja enviar os itens que existem atualmente no Bucket, visto que ele apenas fará com novos itens a partir que esse processo foi estabelecido

#### 4. Opção Extra

Opção de ativar o Delete Marker Replication, ou seja, ele vai replicar itens marcados com deletion pro bucket destino

![[Pasted image 20260408133434.png]]