
Lifecycle Rules são políticas que automatizam a **transição entre Storage Classes** ou a **expiração (deleção) de objetos** com base na idade ou em atributos do objeto.

Sem Lifecycle Rules, objetos permanecem na classe original para sempre — gerando custo desnecessário conforme os dados envelhecem e são acessados com menos frequência.

## Índice

- [[#Tipos de Ação]]
- [[#Transition Actions]]
- [[#Expiration Actions]]
- [[#Filtros de Escopo]]
- [[#Lifecycle Rules e Versionamento]]
- [[#Exemplo: Data Lake com Lifecycle]]
- [[#Onde Configurar]]
- [[#Boas Práticas]]

---

## Tipos de Ação

| Ação | O que faz |
|---|---|
| **Transition Actions** | Move o objeto para outra Storage Class após N dias |
| **Expiration Actions** | Deleta o objeto (ou versões/markers) após N dias |

---

## Transition Actions

Define após quantos dias um objeto deve ser movido para uma classe mais barata.

Fluxo típico de transição:

```
Upload → Standard
  └── após 30 dias  → Standard-IA
        └── após 90 dias  → Glacier Instant Retrieval
              └── após 180 dias → Glacier Deep Archive
                    └── após 365 dias → Expirar (deletar)
```

> Restrições de transição:
> - Só é possível transitar para classes **mais baratas** (sentido único — não volta)
> - Objetos precisam ficar no mínimo **30 dias em Standard** antes de transitar para Standard-IA ou One Zone-IA
> - O tempo mínimo de armazenamento de cada classe é respeitado (ex: Glacier = 90 dias)

---

## Expiration Actions

Deleta objetos automaticamente após um período definido. Pode ser aplicado a:

| Alvo | Descrição |
|---|---|
| **Objetos correntes** | Deleta a versão atual após N dias |
| **Versões não-correntes** | Deleta versões antigas (quando versionamento está ativo) |
| **Delete Markers expirados** | Remove Delete Markers que não têm mais nenhuma versão associada |
| **Uploads multipart incompletos** | Limpa partes de uploads que nunca foram finalizados |

> Uploads multipart abandonados acumulam custo silenciosamente — configure expiração de multipart incompletos em todos os buckets de ingestão.

---

## Filtros de Escopo

Uma Lifecycle Rule pode ser aplicada a:

- **Todo o bucket** — sem filtro
- **Prefixo específico** — ex: `raw/source=crm/` para afetar só aquela partição
- **Tags de objeto** — ex: `env=prod` ou `retention=short`
- **Tamanho do objeto** — mínimo e/ou máximo em bytes (útil para ignorar arquivos muito pequenos)
- Combinação de prefixo + tags + tamanho

---

## Lifecycle Rules e Versionamento

Quando o versionamento está habilitado, as regras podem ser aplicadas separadamente para versões correntes e não-correntes:

```
Versão corrente:
  └── após 90 dias → Standard-IA

Versões não-correntes:
  └── após 30 dias → Glacier Flexible Retrieval
  └── após 90 dias → Expirar

Delete Markers sem versão vinculada:
  └── Expirar automaticamente (evita acúmulo de markers órfãos)
```

---

## Exemplo: Data Lake com Lifecycle

Estrutura de regras para um Data Lake típico:

| Prefixo | Ação | Dias | Destino |
|---|---|---|---|
| `raw/` | Transition | 60 | Standard-IA |
| `raw/` | Transition | 180 | Glacier Flexible |
| `raw/` | Expiration | 730 | Deletar |
| `trusted/` | Transition | 90 | Standard-IA |
| `trusted/` | Transition | 365 | Glacier Instant Retrieval |
| `refined/` | Sem regra | — | Mantém em Standard |
| `compliance/` | Transition | 30 | Glacier Deep Archive |
| `compliance/` | Expiration | 3650 | Deletar (10 anos) |

---

## Onde Configurar

No console: `S3 > Bucket > Management > Lifecycle rules > Create lifecycle rule`

Também pode ser definido via:
- **AWS CLI**: `aws s3api put-bucket-lifecycle-configuration`
- **CloudFormation / Terraform**: recurso `AWS::S3::Bucket` com propriedade `LifecycleConfiguration`
- **AWS SDK**

---

## Boas Práticas

- Sempre configure **expiração de multipart uploads incompletos** (ex: após 7 dias) em buckets de ingestão
- Use **prefixos por zona** (raw/, trusted/, refined/) para aplicar regras distintas por camada do Data Lake
- Combine com **versionamento** e expire versões não-correntes para evitar acúmulo de custo invisível
- Use **S3 Storage Lens** para monitorar qual fração do armazenamento está em cada classe e identificar oportunidades de otimização
- Em buckets com objetos muito pequenos, avalie se o custo de monitoramento do **Intelligent-Tiering** supera o ganho — nesses casos, Lifecycle Rules manuais são mais eficientes


![[Pasted image 20260408143646.png]]
