
## Índice

- [[#O que são e por que existem]]
- [[#Como funcionam internamente]]
- [[#Eventos Disponíveis]]
- [[#Destinos e Quando Usar Cada Um]]
  - [[#SNS — Simple Notification Service]]
  - [[#SQS — Simple Queue Service]]
  - [[#Lambda]]
  - [[#EventBridge]]
- [[#Caso de Uso Clássico]]
- [[#IAM Permissions — Resource Policies]]
- [[#S3 Event Notifications com Amazon EventBridge]]
- [[#Hands On]]

---

## O que são e por que existem

Sem Event Notifications, o S3 é **passivo**: você armazena e recupera arquivos, mas ninguém sabe que um arquivo chegou — a menos que algum sistema vá lá verificar ativamente (polling).

Polling é custoso e lento: você precisa de um processo rodando continuamente perguntando "chegou algo novo?", mesmo quando não chegou nada. Em pipelines de dados isso significa latência desnecessária e desperdício de recursos.

Event Notifications resolvem isso com o modelo **event-driven (orientado a eventos)**: o próprio S3 avisa os sistemas interessados no momento em que algo acontece. Ninguém precisa ficar "olhando" — o S3 empurra a informação.

---

## Como funcionam internamente

Quando uma operação ocorre em um objeto (upload, deleção, restore, etc.), o S3:

1. Detecta a operação internamente
2. Verifica se existe alguma Notification Rule configurada para aquele evento + prefixo/sufixo
3. Se sim, publica uma **mensagem JSON** no destino configurado (SNS, SQS, Lambda ou EventBridge)
4. O destino recebe a mensagem e reage

A mensagem entregue contém informações sobre o evento:

```json
{
  "Records": [{
    "eventName": "ObjectCreated:Put",
    "eventTime": "2025-04-08T14:30:00.000Z",
    "s3": {
      "bucket": { "name": "my-datalake" },
      "object": {
        "key": "raw/source=crm/year=2025/month=04/day=08/file.parquet",
        "size": 4823201,
        "eTag": "abc123..."
      }
    }
  }]
}
```

> Esse JSON é o que chega no Lambda, SQS ou SNS. O sistema receptor sabe exatamente qual arquivo chegou, em qual bucket, e quando — sem precisar fazer nenhuma chamada adicional ao S3.

**Entrega:** geralmente em segundos, mas sem garantia de ordem e com possibilidade de entrega duplicada em casos raros (o destino deve ser idempotente).

---

## Eventos Disponíveis

| Evento | Quando é disparado |
|---|---|
| `s3:ObjectCreated` | Upload, cópia, multipart upload completo |
| `s3:ObjectRemoved` | Deleção de objeto ou versão |
| `s3:ObjectRestore` | Início ou conclusão de restore do Glacier |
| `s3:Replication` | Falha, atraso ou ausência de réplica |
| `s3:LifecycleTransition` | Transição de Storage Class via Lifecycle Rule |
| `s3:IntelligentTiering` | Mudança automática de tier |

Filtros disponíveis por regra:
- **Prefixo** — ex: `raw/source=crm/` (só arquivos naquela partição)
- **Sufixo** — ex: `.parquet`, `.csv`, `.jpg`
- Combinação de ambos

---

## Destinos e Quando Usar Cada Um

```
Amazon S3
    │
    ├──► SNS        → fan-out: distribui o mesmo evento para múltiplos assinantes
    ├──► SQS        → fila de processamento: desacopla produtor do consumidor
    ├──► Lambda     → processamento direto, serverless, sem infra
    └──► EventBridge → roteamento avançado para 18+ serviços
```

### SNS — Simple Notification Service
**Quando usar:** quando o mesmo evento precisa chegar a múltiplos destinos diferentes ao mesmo tempo (fan-out).

**Como funciona na prática:**
- S3 publica uma mensagem no tópico SNS
- O tópico distribui para todos os seus assinantes simultaneamente (ex: um SQS + um email + outro Lambda)
- Útil quando o mesmo arquivo precisa acionar tanto um pipeline de processamento quanto um alerta de monitoramento

**Exemplo real:** arquivo chega na zona Raw → SNS notifica simultaneamente (1) job de validação de schema via Lambda e (2) fila SQS de ingestão para o Glue

### SQS — Simple Queue Service
**Quando usar:** quando o processamento pode ser assíncrono e você quer controlar o ritmo de consumo — especialmente quando chegam muitos arquivos ao mesmo tempo.

**Como funciona na prática:**
- S3 envia a mensagem para a fila
- Um ou mais consumers (Lambda, ECS, EC2, Glue) leem as mensagens no seu próprio ritmo
- Se o consumer falhar, a mensagem fica na fila e pode ser reprocessada
- Protege o sistema downstream de picos de ingestão

**Exemplo real:** 500 arquivos chegam simultaneamente na zona Raw → todos vão para o SQS → workers do Glue processam 10 por vez, sem sobrecarregar o cluster

### Lambda
**Quando usar:** quando você quer executar código imediatamente ao chegar um arquivo, sem manter infraestrutura rodando.

**Como funciona na prática:**
- S3 invoca a função diretamente, passando o JSON do evento
- Lambda executa, processa, e encerra
- Ideal para operações rápidas: validação, transformação leve, trigger de outro serviço

**Exemplo real:** arquivo `.csv` chega no bucket → Lambda valida o schema, converte para Parquet e salva na zona Trusted; se falhar, envia alerta para o SNS

### EventBridge

**Quando usar:** quando você precisa de lógica de roteamento complexa, múltiplos destinos, ou quer arquivar e reprocessar eventos.

**Como funciona na prática:**
- S3 envia todos os eventos para o EventBridge
- Você cria **regras** que filtram eventos por qualquer atributo (tipo, prefixo, tamanho, tag, conta origem)
- Cada regra roteia para um ou mais destinos diferentes
- Eventos podem ser arquivados e reexecutados (replay) — útil para reprocessar dados históricos sem re-fazer uploads

**Exemplo real:** mesmo evento `ObjectCreated` aciona simultaneamente Step Functions (pipeline de ETL), Kinesis Data Streams (para streaming analytics) e um log no CloudWatch — tudo via uma única regra no EventBridge

---

## Caso de Uso Clássico

Gerar thumbnails automaticamente ao fazer upload de imagens no S3:

![[Pasted image 20260408144757.png]]

---

## IAM Permissions — Resource Policies

Para que o S3 possa publicar eventos nesses destinos, **cada serviço de destino precisa ter uma Resource Policy** autorizando o S3 a escrever nele. Não é uma IAM Role no S3 — é uma policy no recurso de destino.

**Por que é assim?** O S3 age como um serviço da AWS chamando outro serviço. Para que isso seja seguro, o destino precisa declarar explicitamente: "aceito receber chamadas do serviço S3, vindo deste bucket específico". Sem isso, o S3 seria barrado.

![[Pasted image 20260408144919.png]]

A imagem mostra os três modelos de Resource Policy — um para cada destino:

**SNS Resource Policy** — permite que o S3 publique no tópico:
```json
{
  "Effect": "Allow",
  "Action": "SNS:Publish",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Resource": "arn:aws:sns:us-east-1:123456789012:MyTopic",
  "Condition": {
    "ArnLike": { "aws:SourceArn": "arn:aws:s3:::MyBucket" }
  }
}
```

**SQS Resource Policy** — permite que o S3 envie mensagens para a fila:
```json
{
  "Effect": "Allow",
  "Action": "SQS:SendMessage",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Resource": "arn:aws:sqs:us-east-1:123456789012:MyQueue",
  "Condition": {
    "ArnLike": { "aws:SourceArn": "arn:aws:s3:::MyBucket" }
  }
}
```

**Lambda Resource Policy** — permite que o S3 invoque a função:
```json
{
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction",
  "Condition": {
    "ArnLike": { "AWS:SourceArn": "arn:aws:s3:::MyBucket" }
  }
}
```

> **Por que o `Condition > ArnLike > aws:SourceArn`?**
> Sem essa condição, a policy estaria dizendo "qualquer bucket S3 da AWS pode acionar este recurso" — o que é um risco de segurança chamado **confused deputy problem**: um atacante poderia criar um bucket próprio e usar sua função Lambda/fila como destino. A condição restringe a permissão ao ARN exato do seu bucket.

---

## S3 Event Notifications com Amazon EventBridge

Uma alternativa mais poderosa às notificações diretas:

```
Amazon S3  ──►  EventBridge  ──►  18+ serviços AWS
```

| Capacidade | Detalhe |
|---|---|
| **Filtragem avançada** | Regras em JSON com filtros por metadata, tags, tamanho, conta origem, etc. |
| **Múltiplos destinos** | Um evento aciona Step Functions, Kinesis, Lambda, SQS e mais — simultaneamente |
| **Archive & Replay** | Arquiva eventos e permite reprocessá-los sem re-fazer uploads |
| **Entrega garantida** | Retry automático com dead-letter queue |
| **Cross-account** | Roteia eventos para recursos em outras contas AWS |

> **Notificação direta** (SNS/SQS/Lambda): simples, baixa latência, menos configuração — ideal para casos diretos com 1 destino
>
> **EventBridge**: quando precisa de múltiplos destinos, filtros complexos, replay de eventos, ou rastreabilidade completa do pipeline

# Hands On

### 1. Criar um Bucket ou editar um existente
### 2. Ir para Properties > Event notifications

![[Pasted image 20260408150343.png]]

### 3. Ativar Amazon EventBridge

![[Pasted image 20260408150424.png]]

### 4. Criar um novo Event notification

![[Pasted image 20260408150506.png]]

#### 4.1 Definir nome e detalhes

![[Pasted image 20260408150545.png]]
### 4.2 Personalizar os Event types

![[Pasted image 20260408150637.png]]

### 4.3 Definir o destino

Nesse caso SQS queues

![[Pasted image 20260408150802.png]]

### 5. Criar Queue

![[Pasted image 20260408150730.png]]

![[Pasted image 20260408150837.png]]

![[Pasted image 20260408150856.png]]

`DemoS3Notification`

### 6. Editar as Access Policies da queue

Para que essa queue possa aceitar notificações do S3, devemos personalizar o JSON de Access policy dessa queue criada

![[Pasted image 20260408151224.png]]

#### 6.1 Criar nova policy/editar a existente

Acessar o https://awspolicygen.s3.amazonaws.com/policygen.html e preencher as informações

![[Pasted image 20260408151438.png]]

Principal = `*` todos tem acesso
Actions = `SendMessage`
Amazon Resource Name = resource do JSON da policy da queue

![[Pasted image 20260408151547.png|412]]

#### 6.2 Copiar nova policy e sobrescrever a policy atual da queue

![[Pasted image 20260408151707.png]]

### 7. Selecionar o SQS que criamos no Bucket

![[Pasted image 20260408152044.png]]

### 8. Fazer o teste de notificação
#### 8.1 Upload de algum arquivo

![[Pasted image 20260408152135.png]]

#### 8.2 Checar no SQS

Entrar em Send and receive messages

![[Pasted image 20260408152200.png]]

Clicar em pool for messages

![[Pasted image 20260408152240.png]]

Podemos observar que a chave da mensagem é justamente o arquivo que salvamos no bucket

`key":"WhatsApp+Image+2025-12-03+at+11.21.42.jpeg"`

![[Pasted image 20260408152254.png]]