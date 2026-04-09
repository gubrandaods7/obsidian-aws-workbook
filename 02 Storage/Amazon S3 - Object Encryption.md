
O Amazon S3 suporta quatro métodos de criptografia para proteger objetos em repouso e em trânsito. A criptografia server-side é habilitada por padrão desde 2023 (SSE-S3).

## Índice

- [[#Conceitos Fundamentais]]
- [[#Comparativo Geral]]
- [[#Server-Side Encryption (SSE)]]
  - [[#1. SSE-S3 — Chaves Gerenciadas pela AWS (Padrão)]]
  - [[#2. SSE-KMS — Chaves Gerenciadas pelo AWS KMS]]
  - [[#3. SSE-C — Chaves Fornecidas pelo Cliente]]
- [[#Client-Side Encryption]]
- [[#Encryption in Transit (SSL/TLS)]]
- [[#Default Encryption vs. Bucket Policies]]
- [[#Boas Práticas para Data Engineering]]
- [[#Hands-on]]

---

## Conceitos Fundamentais

Dados podem ser interceptados ou comprometidos em dois momentos distintos:

- **At rest (em repouso)** — o arquivo está salvo no disco/servidor
- **In transit (em trânsito)** — o arquivo está sendo transferido pela rede

Cada tipo de criptografia endereça um desses momentos — ou os dois.

### Server-Side vs Client-Side

No **Server-Side Encryption**, você envia o dado em claro pela rede. A AWS recebe e criptografa antes de gravar no disco:

```
Você ──── dado legível ────► AWS recebe ──► [criptografa] ──► disco 🔒
```

No **Client-Side Encryption**, você criptografa antes de enviar. A AWS nunca vê o dado em claro — nem na rede, nem no disco:

```
Você ──► [criptografa] ──── dado criptografado ────► AWS ──► disco 🔒
```

### Encryption in Transit (HTTPS)

É uma camada **ortogonal** às anteriores — protege o **canal de comunicação**, não o dado armazenado.

```
HTTP  → canal aberto:    qualquer um na rede pode ler o conteúdo
HTTPS → canal com TLS:   conteúdo ilegível mesmo se interceptado
```

> HTTPS protege o dado *durante o trajeto*. O que acontece depois que ele chega no servidor é responsabilidade do SSE — as duas proteções são independentes e complementares.

### Como as camadas se combinam

| Cenário | At Rest | In Transit |
|---|---|---|
| HTTP + sem criptografia | Dado legível no disco | Dado legível na rede |
| HTTPS + sem criptografia | Dado legível no disco | Dado protegido na rede |
| HTTP + SSE-S3 | Dado protegido no disco | Dado legível na rede |
| **HTTPS + SSE-KMS** | **Dado protegido no disco** | **Dado protegido na rede** |

---

## Comparativo Geral

| Método | Onde ocorre | Quem gerencia a chave | HTTPS obrigatório |
|---|---|---|---|
| **SSE-S3** | Servidor (AWS) | AWS | Não |
| **SSE-KMS** | Servidor (AWS) | AWS KMS (com controle do usuário) | Não |
| **SSE-C** | Servidor (AWS) | Cliente (fora da AWS) | **Sim** |
| **Client-Side** | Cliente | Cliente (totalmente) | Recomendado |

---

# Server-Side Encryption (SSE)

## 1. SSE-S3 — Chaves Gerenciadas pela AWS (Padrão)

- Criptografia realizada com chaves criadas, gerenciadas e de propriedade da AWS
- Algoritmo: **AES-256**
- Header HTTP necessário: `"x-amz-server-side-encryption": "AES256"`
- **Habilitado por padrão** para todos os novos objetos e buckets desde 2023

![[Pasted image 20260409092737.png]]

---

## 2. SSE-KMS — Chaves Gerenciadas pelo AWS KMS

- Criptografia realizada com chaves gerenciadas pelo **AWS KMS (Key Management Service)**
- O usuário cria e controla as chaves, com **auditoria completa via CloudTrail**
  - Cada uso da chave gera um log no CloudTrail — útil para compliance
- Header HTTP necessário: `"x-amz-server-side-encryption": "aws:kms"`

![[Pasted image 20260409093027.png]]

### Limitações de Throughput (KMS Throttling)

- Cada upload chama a API **`GenerateDataKey`** do KMS
- Cada download chama a API **`Decrypt`** do KMS
- Essas chamadas consomem cotas do KMS por segundo:

| Região | Limite padrão (req/s) |
|---|---|
| us-east-1, us-west-2, eu-west-1 | 30.000 |
| Outras regiões | 5.500 – 10.000 |

> Cotas podem ser aumentadas via **Service Quotas Console**. Em jobs de leitura massiva (Spark/EMR), o throttling do KMS pode se tornar um gargalo real.

![[Pasted image 20260409093351.png]]

---

## 3. SSE-C — Chaves Fornecidas pelo Cliente

- Criptografia server-side com chaves gerenciadas pelo usuário **fora da AWS**
- A AWS usa a chave para criptografar/descriptografar, mas **não a armazena**
- **HTTPS é obrigatório** — a chave é transmitida no header HTTP e não pode ser exposta
- A chave deve ser enviada em **todas as requisições** (upload e download)

![[Pasted image 20260409093703.png]]

---

# Client-Side Encryption

- O cliente criptografa os dados **antes** de enviar ao Amazon S3
- O cliente descriptografa os dados **após** receber do Amazon S3
- O usuário gerencia por completo as chaves e o ciclo de criptografia
- Biblioteca de referência: **Amazon S3 Client-Side Encryption Library**

![[Pasted image 20260409093944.png]]

---

# Encryption in Transit (SSL/TLS)

O Amazon S3 expõe dois endpoints:

| Endpoint | Tipo |
|---|---|
| `http://` | Não criptografado |
| `https://` | Criptografado em trânsito (TLS) |

- HTTPS é **recomendado** para todos os casos
- HTTPS é **obrigatório** para SSE-C
- Para **forçar HTTPS** em todo o bucket, use uma Bucket Policy com a condição `aws:SecureTransport`:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": {
    "Bool": { "aws:SecureTransport": "false" }
  }
}
```

![[Pasted image 20260409094229.png]]

---

# Default Encryption vs. Bucket Policies

## Criptografia Padrão (Default Encryption)

A criptografia padrão do bucket (SSE-S3) é **aplicada automaticamente** a qualquer novo objeto armazenado — não é preciso fazer nada. É conveniente, mas tem uma limitação: ela não **impede** que alguém tente fazer upload com um tipo diferente de criptografia (ou sem criptografia no header).

## Forçar Criptografia via Bucket Policy

Para **garantir** que nenhum objeto seja armazenado fora do padrão exigido, usa-se uma Bucket Policy para **recusar** chamadas `PutObject` que não incluam o header de criptografia correto.

> **Atenção:** Bucket Policies são avaliadas **antes** da criptografia padrão ser aplicada. Se a policy negar o request, ele falha — a criptografia default não entra em ação para "salvar" o upload.

### Forçar SSE-KMS

Nega qualquer upload que não declare explicitamente `aws:kms` no header:

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

### Forçar SSE-C

Nega qualquer upload que não inclua o header do algoritmo de criptografia do cliente (`s3:x-amz-server-side-encryption-customer-algorithm`):

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "Null": {
      "s3:x-amz-server-side-encryption-customer-algorithm": "true"
    }
  }
}
```

> A condição `Null: true` significa "se o header estiver ausente (null) → negar". Ou seja, só permite o upload se o header estiver presente.

### Fluxo de avaliação

```
PutObject request
       │
       ▼
  Bucket Policy ──► DENY? ──► request rejeitado (4xx)
       │
      (allow)
       ▼
  Default Encryption aplicada (se nenhum header foi enviado)
       │
       ▼
  Objeto gravado no disco 🔒
```

---

## Boas Práticas para Data Engineering

- Usar **SSE-KMS** em dados sensíveis (PII, dados financeiros) para auditoria de acesso via CloudTrail
- Monitorar o throughput de chamadas KMS em jobs Spark/EMR para evitar throttling
- Em jobs de leitura massiva, avaliar se SSE-S3 não é suficiente — tem zero overhead de API
- Aplicar **Bucket Policy com `aws:SecureTransport`** para garantir que nenhum dado trafegue via HTTP
- SSE-C raramente é usado em Data Engineering — o overhead operacional de gerenciar chaves externas é alto

# Hands-on

`# gustavo-brandao-demo-encryption-v1`

Criar bucket e habilitar SSE-S3
![[Pasted image 20260409095136.png]]

Subir arquivo e verificar se tem a criptografia
![[Pasted image 20260409095255.png]]

Editar modo de criptografia
Ele vai criar uma nova versão (versioning) dos arquivos existentes

![[Pasted image 20260409095421.png]]

Selecionar chave padrão já existente

![[Pasted image 20260409095533.png]]

Nova versão considera o novo formato de criptografia de SSE-KMS

![[Pasted image 20260409095644.png]]

Ao fazer upload de um novo arquivo, temos opções pra configurar a criptografia

![[Pasted image 20260409095815.png]]



