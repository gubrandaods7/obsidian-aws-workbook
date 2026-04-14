
EFS (Elastic File System) é um serviço de **sistema de arquivos compartilhado na rede** da AWS. Diferente do EBS (que é um disco exclusivo de uma instância), o EFS pode ser montado em **múltiplas instâncias EC2 simultaneamente**, funcionando como uma pasta de rede compartilhada acessível por vários servidores ao mesmo tempo.

## Índice

- [[#Características Principais]]
- [[#Como o EFS Funciona]]
- [[#Performance Mode]]
- [[#Throughput Mode]]
- [[#Storage Tiers]]
- [[#Availability and Durability]]
- [[#Casos de Uso]]

---

## Características Principais

- **Filesystem compartilhado**: múltiplas instâncias EC2 podem montar o mesmo volume simultaneamente — algo que o EBS padrão não permite
- **Multi-AZ nativo**: o EFS opera automaticamente em várias Availability Zones, diferente do EBS que fica preso a uma única AZ
- **Escala automática**: cresce e encolhe conforme os dados são adicionados ou removidos — sem necessidade de provisionar capacidade
- **Alta disponibilidade e durabilidade**: dados replicados entre zonas por padrão
- **Apenas Linux**: o EFS usa o protocolo NFS e **não é compatível com Windows**
- **Mais caro que o EBS**: o custo por GB é maior, mas você paga apenas pelo que usa (sem pre-provisioning)

![[Pasted image 20260414130928.png]]

---

## Como o EFS Funciona

O EFS usa o protocolo **NFS (Network File System)** — um padrão de rede para acesso remoto a arquivos, semelhante a uma pasta compartilhada em rede corporativa.

**Implicações práticas:**
- O acesso é controlado por **Security Groups** — a instância EC2 precisa ter a porta NFS (2049) liberada para o EFS
- Por ser tráfego de rede, pode haver **latência** maior que armazenamento local ou EBS gp3 em cargas com muitas operações pequenas
- Os dados são **criptografados em repouso** (at-rest) por padrão usando AWS KMS
- O sistema de arquivos aparece como um **diretório Linux comum** — sem diferença de uso para a aplicação

---

## Performance Mode

Define a característica de latência e throughput do sistema de arquivos.

| Modo | Latência | Paralelismo | Quando usar |
|---|---|---|---|
| **General Purpose** | Menor | Moderado | Maioria dos casos — web servers, CMS, aplicações que precisam de respostas rápidas |
| **Max I/O** | Maior | Muito alto | Cargas massivamente paralelas — big data, processamento de mídia, análise de dados |

> **Regra prática**: comece com General Purpose. Só mude para Max I/O se o throughput for o gargalo com muitos clientes simultâneos.

---

## Throughput Mode

Define a quantidade de dados por segundo (MB/s) que pode ser lida ou escrita.

| Modo | Como funciona | Quando usar |
|---|---|---|
| **Bursting** | Throughput proporcional ao tamanho do filesystem — acumula créditos em períodos ociosos | Uso com picos imprevisíveis e filesystem razoavelmente grande |
| **Provisioned** | Velocidade definida manualmente, independente do tamanho | Throughput alto necessário com filesystem pequeno |
| **Elastic** | Ajusta automaticamente conforme a demanda — sem gerenciamento | Uso imprevisível ou variável — recomendado para a maioria dos casos novos |

---

## Storage Tiers

O EFS possui níveis de armazenamento com diferentes custos e velocidades de acesso. O **Lifecycle Management** move arquivos automaticamente entre tiers com base em tempo de último acesso.

| Tier | Acesso | Custo | Quando usar |
|---|---|---|---|
| **Standard** | Frequente | Mais alto | Arquivos ativos acessados regularmente |
| **Infrequent Access (IA)** | Raro | Reduzido | Arquivos acessados esporadicamente |
| **Archive** | Muito raro | Muito baixo | Arquivos históricos, backups antigos, compliance |

### Lifecycle Policy

O EFS move os arquivos automaticamente:

- Após **X dias sem acesso** → promove para Infrequent Access
- Após mais tempo sem acesso → promove para Archive
- Se acessado novamente → pode retornar ao Standard automaticamente

---

## Availability and Durability

| Arquitetura | Replicação | Custo | Quando usar |
|---|---|---|---|
| **Standard (Regional)** | Múltiplas AZs | Mais alto | Produção — alta disponibilidade e tolerância a falhas de zona |
| **One Zone** | Única AZ | ~47% mais barato | Dev/test, cargas não críticas, quando os dados são reproduzíveis |

### Comportamento padrão

- A arquitetura **Standard** é a padrão ao criar um novo EFS
- One Zone não oferece a mesma durabilidade — perda da AZ implica perda de dados

---

## Casos de Uso

- **CMS e sites** (WordPress, Drupal) — múltiplos servidores web compartilhando os mesmos assets
- **Servidores de aplicação horizontalmente escalados** — instâncias EC2 em Auto Scaling acessando os mesmos arquivos
- **Compartilhamento de dados entre ambientes** — dev, staging e prod lendo do mesmo filesystem
- **HPC e computação científica** — software legado (simulações, genômica) que usa chamadas POSIX (`open`, `seek`, `flock`) e não pode ser reescrito para usar a API do S3; para big data moderno (Spark, Athena), prefira S3 ou S3 Tables
- **Backup e armazenamento de longo prazo** — com Archive tier para dados raramente acessados
