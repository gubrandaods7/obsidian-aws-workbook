
Ao criar um objeto no S3, você escolhe (ou o S3 gerencia automaticamente) em qual Storage Class ele será armazenado. A escolha impacta diretamente custo, latência de acesso e disponibilidade.

---

## Durability vs Availability

| Conceito | O que mede | Valor |
|---|---|---|
| **Durability** | Probabilidade de não perder o objeto | **99,999999999%** ("11 noves") — igual em todas as classes |
| **Availability** | Fração do tempo em que o objeto está acessível | **Varia por classe** |

> Durabilidade de 11 noves: em 10 milhões de objetos armazenados, a expectativa é perder 1 objeto a cada 10.000 anos.

> Disponibilidade de 99,99% = até **53 minutos indisponíveis por ano**.

---

## Tabela Comparativa

| Storage Class | Disponibilidade | Armazenamento mínimo | Retrieval | Custo relativo |
|---|---|---|---|---|
| **Standard** | 99,99% | Sem mínimo | Imediato (ms) | Alto |
| **Intelligent-Tiering** | 99,9% | Sem mínimo | Imediato (ms) | Médio + taxa de monitoramento |
| **Standard-IA** | 99,9% | 30 dias | Imediato (ms) | Médio-baixo + taxa de retrieval |
| **One Zone-IA** | 99,5% | 30 dias | Imediato (ms) | Baixo + taxa de retrieval |
| **Glacier Instant Retrieval** | 99,9% | 90 dias | Imediato (ms) | Muito baixo + taxa de retrieval |
| **Glacier Flexible Retrieval** | 99,99% | 90 dias | 1 min a 12 h | Muito baixo + taxa de retrieval |
| **Glacier Deep Archive** | 99,99% | 180 dias | 12 h a 48 h | Mínimo + taxa de retrieval |

> **Taxa de retrieval**: classes IA e Glacier cobram por GB recuperado, além da taxa de armazenamento. Considere isso se os dados forem lidos com alguma frequência.

---

## S3 Standard — General Purpose

- Dados acessados **frequentemente**
- Latência em milissegundos, throughput alto
- Replicado em no mínimo **3 AZs** — suporta até 2 falhas simultâneas sem perda de dados

**Casos de uso:** Big Data analytics, aplicações em produção, distribuição de conteúdo, jogos, aplicações mobile

---

## S3 Standard-IA (Infrequent Access)

- Dados acessados com **baixa frequência**, mas que precisam de acesso rápido quando necessário
- Mesmo desempenho de latência que o Standard
- Custo de armazenamento menor, mas cobra **taxa por retrieval** e tem **mínimo de 30 dias**
- 99,9% de disponibilidade

**Casos de uso:** Disaster Recovery, backups que precisam ser restaurados rapidamente, dados históricos de Data Lakes

---

## S3 One Zone-IA

- Mesmo propósito do Standard-IA, mas armazenado em **apenas 1 AZ**
- Se a AZ for destruída, os dados são **perdidos permanentemente**
- 99,5% de disponibilidade — menor custo entre as classes de acesso imediato
- Durabilidade de 99,999999999% **dentro** da AZ

**Casos de uso:** Backups secundários de dados on-premise que podem ser recriados, dados de baixa criticidade, thumbnails/caches regeneráveis

---

## S3 Glacier Instant Retrieval

- Arquivamento com recuperação em **milissegundos** — mesma latência do Standard, com custo de Glacier
- Ideal para dados acessados raramente (ex: uma vez por trimestre), mas que não toleram espera
- Mínimo de **90 dias** de armazenamento

**Casos de uso:** Imagens médicas, registros de compliance acessados ocasionalmente, dados de auditoria

---

## S3 Glacier Flexible Retrieval

- Arquivamento de custo baixo com **3 opções de velocidade**:

| Opção | Tempo | Custo de retrieval |
|---|---|---|
| **Expedited** | 1 a 5 minutos | Cobrado por GB + request |
| **Standard** | 3 a 5 horas | Cobrado por GB + request |
| **Bulk** | 5 a 12 horas | Gratuito |

- Mínimo de **90 dias** de armazenamento

**Casos de uso:** Backups que toleram horas de espera, arquivos de projetos encerrados, dados históricos raramente consultados

---

## S3 Glacier Deep Archive

- A classe **mais barata** do S3 — voltada para retenção de longa duração
- Duas opções de retrieval:

| Opção | Tempo |
|---|---|
| **Standard** | até 12 horas |
| **Bulk** | até 48 horas |

- Mínimo de **180 dias** de armazenamento

**Casos de uso:** Dados de compliance com retenção de 7-10 anos, substituição de fitas magnéticas, arquivos legais ou fiscais exigidos por regulação

---

## S3 Intelligent-Tiering

Move objetos **automaticamente** entre tiers com base no padrão de acesso — sem intervenção manual.

| Tier | Ativação | Tipo |
|---|---|---|
| **Frequent Access** | Default | Automático |
| **Infrequent Access** | Sem acesso por 30 dias | Automático |
| **Archive Instant Access** | Sem acesso por 90 dias | Automático |
| **Archive Access** | 90 a 700+ dias (configurável) | Opcional |
| **Deep Archive Access** | 180 a 700+ dias (configurável) | Opcional |

- **Não cobra taxa de retrieval** entre os tiers automáticos
- Cobra **taxa de monitoramento mensal por objeto** — relevante para grandes volumes de objetos pequenos
- Sem mínimo de permanência por tier

**Ideal quando:** o padrão de acesso é imprevisível ou varia ao longo do tempo

---

![[Pasted image 20260408141354.png]]![[Pasted image 20260408141408.png]]

---

## Como Escolher a Storage Class

```
Acesso frequente?
  └── Sim → Standard

Acesso imprevisível ou variável?
  └── Sim → Intelligent-Tiering

Acesso infrequente, mas recuperação rápida?
  ├── Dados críticos (multi-AZ) → Standard-IA
  └── Dados recriáveis (1 AZ)   → One Zone-IA

Arquivamento com recuperação em ms?
  └── Glacier Instant Retrieval

Arquivamento tolerando horas de espera?
  ├── Acesso eventual          → Glacier Flexible Retrieval
  └── Retenção longa, raro    → Glacier Deep Archive
```

---

## Contexto em Data Lakes

| Zona | Storage Class recomendada | Justificativa |
|---|---|---|
| **Raw / Landing** | Standard ou Intelligent-Tiering | Dados novos acessados com frequência em ingestão e validação |
| **Trusted / Curated** | Standard-IA (dados > 30 dias) | Lidos em jobs analíticos periódicos, não contínuos |
| **Refined / Serving** | Standard | Camada de consumo por BI/dashboards — acesso frequente |
| **Compliance / Auditoria** | Glacier Flexible ou Deep Archive | Acesso raro, retenção longa exigida por regulação |

> Combine Storage Classes com **Lifecycle Policies** para transição automática conforme os dados envelhecem — evita gestão manual e reduz custo progressivamente.
