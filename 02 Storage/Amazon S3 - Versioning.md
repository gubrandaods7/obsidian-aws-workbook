
O versionamento permite manter múltiplas versões de um mesmo objeto dentro de um bucket.

- Habilitado no **nível do bucket**
- Uma vez habilitado, **não pode ser totalmente desabilitado** — apenas suspenso
- Quando suspenso, versões anteriores são preservadas; novos uploads recebem version ID `null`

![[Pasted image 20260408103725.png]]

## Índice

- [[#Por que usar Versionamento?]]
- [[#Como funciona]]
- [[#Objetos anteriores ao versionamento]]
- [[#Suspensão do Versionamento]]
- [[#Integração com Lifecycle Policies]]
- [[#Integração com Replicação]]
- [[#Boas Práticas]]
- [[#Hands-on - Versioning]]

---

## Por que usar Versionamento?

| Benefício | Detalhe |
|---|---|
| **Proteção contra deleção acidental** | Deletar um objeto cria um Delete Marker em vez de apagar de verdade |
| **Proteção contra sobrescrita** | Cada PUT gera uma nova versão; a anterior é preservada |
| **Rollback fácil** | É possível restaurar qualquer versão anterior de um objeto |
| **Auditoria** | Histórico completo de alterações em um arquivo |

---

## Como funciona

### Upload de novas versões

Cada vez que um objeto é enviado com a mesma chave (key), o S3 cria uma nova versão com um **Version ID único**. A versão mais recente é retornada por padrão em operações de leitura.

```
s3://my-bucket/report.csv  →  Version ID: abc123  (mais recente)
s3://my-bucket/report.csv  →  Version ID: xyz789  (versão anterior)
s3://my-bucket/report.csv  →  Version ID: null    (antes do versionamento ser habilitado)
```

### Deleção com versionamento ativo

- Deletar um objeto **sem especificar Version ID** → S3 insere um **Delete Marker** (o objeto some da listagem, mas não foi apagado fisicamente)
- Deletar o **Delete Marker** → objeto "ressurge" (restauração)
- Deletar uma **versão específica** → aquela versão é apagada permanentemente

> Para apagar um objeto de verdade, é necessário deletar cada versão individualmente (incluindo o Delete Marker).

---

## Objetos anteriores ao versionamento

- Objetos que já existiam no bucket antes do versionamento ser habilitado recebem Version ID `null`
- Esses objetos são preservados normalmente; o `null` é tratado como uma versão válida

---

## Suspensão do Versionamento

- Novas versões deixam de ser criadas; novos uploads recebem Version ID `null`
- Versões existentes **são mantidas** e continuam acessíveis
- Não é uma forma de "apagar o histórico" — para isso é preciso deletar as versões manualmente ou via Lifecycle Policy

---

## Integração com Lifecycle Policies

Versioning + Lifecycle é uma combinação poderosa para gerenciar custo:

```
Versões não-correntes após 30 dias   → mover para Standard-IA
Versões não-correntes após 90 dias   → mover para Glacier
Versões não-correntes após 365 dias  → expirar (deletar)
Delete Markers sem versão vinculada  → expirar automaticamente
```

---

## Integração com Replicação

- Versionamento é **obrigatório** em ambos os buckets (origem e destino) para habilitar replicação (CRR/SRR)
- Delete Markers podem ou não ser replicados — configurável separadamente

---

## Boas Práticas

- Habilite versionamento em buckets com dados críticos ou que sofrem sobrescrita frequente
- Use Lifecycle Policies para controlar o custo das versões antigas
- Em Data Lakes, versione a camada **Trusted/Curated** onde os dados já foram processados — na Raw layer, dados costumam ser imutáveis por design
- Monitore o número de versões com **S3 Storage Lens** ou **S3 Inventory** para evitar acúmulo silencioso de dados

---

# Hands-on - Versioning
