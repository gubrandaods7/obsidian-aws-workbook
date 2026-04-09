
Access Points simplificam o gerenciamento de segurança em buckets S3 com muitos prefixos e equipes distintas. 

Em vez de uma única Bucket Policy enorme e complexa, cada grupo de usuários recebe seu próprio ponto de entrada — com sua própria política de acesso.

## Índice

- [[#O Problema que Access Points Resolve]]
- [[#Como Funciona]]
- [[#Estrutura de um Access Point]]
- [[#Exemplo do Slide]]
- [[#VPC Origin]]
- [[#Contexto em Data Lakes]]
- [[#Diagrama — Dois Cenários de Bloqueio]]

---

## O Problema que Access Points Resolve

Em um bucket compartilhado por múltiplas equipes, a Bucket Policy cresce e se torna difícil de gerenciar:

```json
// Bucket Policy monolítica — difícil de manter
{
  "Statement": [
    { "Principal": "finance-team", "Action": "s3:*", "Resource": "arn:.../*finance*" },
    { "Principal": "sales-team",   "Action": "s3:*", "Resource": "arn:.../*sales*" },
    { "Principal": "analytics",    "Action": "s3:Get*", "Resource": "arn:...*" },
    // ... dezenas de outras regras
  ]
}
```

Access Points resolvem isso delegando: cada equipe tem seu próprio endpoint com sua própria policy — menor, focada, fácil de auditar.

---

## Como Funciona

![[Pasted image 20260409124009.png]]

```
Usuários (Finance)    ──► Finance Access Point   ──► /finance/...
                              Policy: R/W no prefixo /finance
                                           │
Usuários (Sales)      ──► Sales Access Point     ──► /sales/...
                              Policy: R/W no prefixo /sales
                                           │           S3 Bucket
Usuários (Analytics)  ──► Analytics Access Point ──► bucket inteiro
                              Policy: Read em todo o bucket
```

Todos os Access Points apontam para o **mesmo bucket S3**. O bucket em si continua existindo normalmente — os Access Points são camadas de acesso sobre ele.

---

## Estrutura de um Access Point

Cada Access Point tem três características básicas:

**1. URL própria** — cada Access Point tem um endereço único para ser chamado. Em vez de chamar o bucket diretamente, a equipe chama o Access Point dela.

**2. Policy própria** — define o que aquele Access Point permite (ex: só leitura, só no prefixo `/finance/`). É como uma Bucket Policy, mas menor e focada em um único grupo de usuários.

**3. Origem** — de onde as chamadas podem vir: internet (padrão) ou somente de dentro de uma VPC.

> O bucket também precisa ter uma Bucket Policy que "autorize" os Access Points a gerenciarem o acesso — sem isso, o Access Point existe mas não consegue chegar no bucket.

---

## Exemplo do Slide

![[Pasted image 20260409130805.png]]

| Access Point | Usuários | Permissão | Escopo |
|---|---|---|---|
| **Finance Access Point** | Equipe de Finanças | Read + Write | Prefixo `/finance/` |
| **Sales Access Point** | Equipe de Vendas | Read + Write | Prefixo `/sales/` |
| **Analytics Access Point** | Equipe de Analytics | Read only | Bucket inteiro |

Cada equipe usa apenas seu próprio endpoint. A equipe de Finanças não consegue acessar `/sales/` — mesmo que tentassem usar o endpoint diretamente, a Access Point Policy bloqueia.

---

## VPC Origin

### O que é uma VPC

**VPC (Virtual Private Cloud)** é uma rede virtual privada dentro da AWS — é como se fosse a "rede interna" da sua empresa, mas hospedada na nuvem.

Pense assim: quando você está no escritório e acessa um sistema interno, você está na rede privada da empresa — ninguém de fora consegue acessar esse sistema diretamente pela internet. 

A VPC funciona da mesma forma: recursos como servidores, jobs de ETL e bancos de dados ficam dentro dessa rede privada, isolados da internet.

**Exemplo prático:** imagine que você tem um job no AWS Glue que lê arquivos do S3, processa e grava o resultado de volta. Esse job nunca precisa da internet — ele só precisa falar com o S3. Ao colocar o Glue dentro de uma VPC e usar um Access Point com VPC 

Origin, você garante que:
- O job acessa o S3 pela rede interna da AWS (mais rápido, mais barato)
- Nenhum dado sai para a internet em nenhum momento
- Se alguém de fora tentar acessar esse Access Point via internet, é bloqueado automaticamente

```
Internet pública
       │
  ─────┼─────────────────────────────
       │         VPC (rede privada)
       │    ┌──────────────────────┐
       │    │  EMR, Glue, Lambda   │
       │    │  RDS, EC2...         │
       │    └──────────────────────┘
```

### Access Points com VPC Origin

Ao criar um Access Point, você escolhe a **origem** dele: pode ser a internet (padrão) ou uma VPC específica.

Quando você escolhe **VPC Origin**, o Access Point passa a só aceitar requisições vindas de dentro daquela VPC — qualquer chamada vinda da internet é automaticamente rejeitada, mesmo que a pessoa tenha as credenciais corretas.

É como dizer: "este ponto de entrada existe, mas só funciona se você estiver dentro da rede interna."

### Fluxo completo com VPC Origin

```
EC2 (ou Glue, EMR...)
       │
       ▼
  VPC Endpoint          ← ponte entre a VPC e o S3 (sem sair para a internet)
  [Endpoint Policy]     ← 1ª camada: define quais buckets/access points este endpoint pode acessar
       │
       ▼
  Access Point
  [Access Point Policy] ← 2ª camada: define quem pode fazer o quê (ex: só leitura no /finance/)
       │
       ▼
  S3 Bucket
  [Bucket Policy]       ← 3ª camada: delegação para os Access Points
```

São **3 camadas de policy** que precisam estar alinhadas para o acesso funcionar:

| Camada | Onde fica | O que controla |
|---|---|---|
| **Endpoint Policy** | No VPC Endpoint | Quais buckets e access points podem ser acessados por quem passa por este endpoint |
| **Access Point Policy** | No Access Point | Quais ações são permitidas e em quais prefixos |
| **Bucket Policy** | No bucket S3 | Delega controle para os Access Points |

### Exemplo de Endpoint Policy (do slide)

```json
{
  "Statement": [{
    "Principal": "*",
    "Action": ["s3:GetObject"],
    "Effect": "Allow",
    "Resource": [
      "arn:aws:s3:::awsexamplebucket1/*",
      "arn:aws:s3:us-west-2:123456789012:accesspoint/example-vpc-ap/object/*"
    ]
  }]
}
```

> A Endpoint Policy precisa liberar acesso tanto ao **bucket** quanto ao **Access Point** — os dois ARNs precisam estar no `Resource`. Se só o bucket estiver liberado, o acesso via Access Point ainda é bloqueado.

**Exemplo prático:** um job no AWS Glue dentro da VPC precisa ler arquivos do S3. O tráfego sai do Glue → passa pelo VPC Endpoint (que tem uma Endpoint Policy permitindo acesso ao bucket e ao access point) → chega no Access Point (que tem uma policy liberando apenas leitura no prefixo correto) → acessa o bucket. Nenhum byte saiu para a internet.

Isso é útil para:
- Jobs de ETL (EMR, Glue) que processam dados e nunca precisam expor nada para a internet
- Compliance que exige que dados sensíveis trafeguem apenas na rede privada
- Redução de custo — tráfego dentro da VPC é mais barato que tráfego pela internet

## Contexto em Data Lakes

Access Points são especialmente úteis quando o Data Lake é compartilhado por múltiplas equipes com diferentes níveis de acesso:

| Equipe | Access Point | Prefixo | Permissão |
|---|---|---|---|
| **Engenharia de Dados** | `de-access-point` | `raw/` + `trusted/` | R/W |
| **Data Science** | `ds-access-point` | `trusted/` + `refined/` | Read |
| **BI / Analysts** | `bi-access-point` | `refined/` | Read |
| **Compliance** | `compliance-access-point` | `compliance/` | Read |

> Vantagem: quando uma equipe muda de responsabilidade ou um colaborador sai, basta atualizar a policy do Access Point correspondente — sem tocar na Bucket Policy central.

---

## Diagrama — Dois Cenários de Bloqueio

![[Pasted image 20260409132610.png]]

O diagrama abaixo mostra que as proteções funcionam em duas direções independentes:

**Cenário 1 — Aplicação dentro da VPC tentando acessar um bucket não autorizado (seta de cima):**

```
App (dentro da VPC) ──► VPC Endpoint ──► S3 Bucket errado
                                              ✗ BLOQUEADO
                              (S3 VPC Endpoint Policy blocks this access)
```

Mesmo estando dentro da VPC, se a **Endpoint Policy** não listar aquele bucket, o acesso é negado. 

Ou seja, a Endpoint Policy não só protege de fora para dentro — ela também restringe o que recursos internos podem acessar. Isso evita que uma aplicação comprometida dentro da VPC acesse buckets que não deveria.

---

**Cenário 2 — Aplicação fora da VPC tentando acessar o bucket diretamente (seta de baixo):**

```
App (fora da VPC) ──► S3 Bucket
                           ✗ BLOQUEADO
                    (S3 Bucket Policy blocks this access)
```

A **Bucket Policy** recusa qualquer acesso direto que não venha pelo Access Point + VPC Endpoint. Mesmo que a aplicação externa tenha credenciais válidas, ela é barrada.

---

**Caminho que funciona:**

```
App (dentro da VPC)
       │
       ▼
  S3 VPC Endpoint  ──►  S3 Access Point  ──►  S3 Bucket ✓
  [Endpoint Policy OK]  [Access Point       [Bucket Policy
                         Policy OK]          delega ao AP]
```

O acesso só é permitido quando os três elementos estão alinhados: a aplicação está dentro da VPC, passa pelo Endpoint correto, e usa o Access Point autorizado.
