
# Tipos de Segurança

## User-Based

**IAM Policies** — definem quais chamadas de API são permitidas para um usuário/role/grupo IAM específico. A policy é atachada à identidade, não ao recurso.

## Resource-Based

**Bucket Policies** — regras em JSON aplicadas diretamente ao bucket. Permitem:
- Acesso público controlado
- Acesso cross-account (outras contas AWS)
- Requisitos como exigir criptografia no upload

**Object ACL (Access Control List)** — controle granular por objeto individual; pode ser desabilitado (recomendado na maioria dos casos modernos)

**Bucket ACL** — controle no nível do bucket; menos comum, também pode ser desabilitado

### Lógica de Avaliação de Acesso

Um usuário IAM pode acessar um objeto do S3 quando:
- A IAM policy do usuário **permite** a ação **OU** a Bucket Policy **permite** a ação
- **E** não existe nenhum `Deny` explícito em nenhuma das políticas

> Se houver qualquer `Deny` explícito, o acesso é bloqueado — independentemente de outros `Allow`.

## Block Public Access

Camada extra de proteção que sobrepõe qualquer Bucket Policy ou ACL:
- **Bloqueia** qualquer configuração que tornaria o bucket ou objetos públicos
- Pode ser habilitado no nível do bucket **ou** no nível da conta AWS inteira
- Deve estar ativado em todos os buckets que não precisam ser públicos (padrão recomendado)

## Encryption

Encripta objetos armazenados no S3 usando chaves de encriptação.

| Tipo | Chave gerenciada por | Auditoria | Caso de uso |
|---|---|---|---|
| **SSE-S3** | AWS (automático) | Não | Padrão; zero configuração |
| **SSE-KMS** | AWS KMS | Sim (CloudTrail) | Dados regulados; trilha de acesso |
| **SSE-C** | Cliente | Não | Cliente controla 100% das chaves |
| **Client-side** | Cliente (antes do upload) | Não | Máxima segurança end-to-end |

> SSE-S3 é habilitado por padrão em todos os novos buckets desde 2023. Para ambientes com compliance (LGPD, PCI, HIPAA), prefira SSE-KMS.

---

# S3 Bucket Policies

São estruturadas em formato JSON com os seguintes campos:

| Campo | Descrição |
|---|---|
| `Resource` | ARN do bucket e/ou objetos afetados (`arn:aws:s3:::my-bucket/*`) |
| `Effect` | `Allow` ou `Deny` |
| `Action` | Conjunto de APIs permitidas/bloqueadas (ex: `s3:GetObject`) |
| `Principal` | Conta, usuário IAM ou `*` (todos) ao qual a policy se aplica |

**Principais usos:**

- Permitir acesso público ao bucket
- Forçar criptografia em todos os uploads (`s3:PutObject` com condição)
- Dar acesso a outra conta AWS (cross-account)
- Restringir acesso a um VPC Endpoint específico

![[Pasted image 20260408102022.png]]

---

## Exemplos — Types of Policy

![[Pasted image 20260408102311.png]]

![[Pasted image 20260408102348.png]]

![[Pasted image 20260408102411.png]]

![[Pasted image 20260408102431.png|696]]

![[Pasted image 20260408102525.png]]

---

## Hands-on — Criando Public Access Policy

![[Pasted image 20260408102641.png]]

![[Pasted image 20260408103154.png]]

> Ferramenta útil para gerar políticas: [AWS Policy Generator](https://awspolicygen.s3.amazonaws.com/policygen.html)

![[Pasted image 20260408103019.png]]

![[Pasted image 20260408103056.png]]

![[Pasted image 20260408103056.png]]

> No campo Resource, use o ARN do bucket seguido de `/*` para aplicar a todos os objetos dentro dele.
> Exemplo: `arn:aws:s3:::nome-do-bucket/*`

![[Pasted image 20260408102955.png]]

![[Pasted image 20260408103326.png]]

Com isso, qualquer objeto/arquivo estará disponível para acesso público.

> **Atenção:** para que a Bucket Policy de acesso público funcione, o **Block Public Access** do bucket precisa estar desativado. Em produção, só faça isso quando realmente necessário (ex: site estático).

