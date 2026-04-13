
EBS (Elastic Block Store) é um serviço de armazenamento em bloco da AWS que funciona como um **disco virtual persistente** conectado a instâncias EC2. Diferente do armazenamento efêmero (Instance Store), os dados no EBS **sobrevivem a reinicializações e desligamentos** da instância.

## Índice

- [[#Características Principais]]
- [[#Como o EBS Funciona]]
- [[#Delete on Termination]]

---

## Características Principais

- **Persistência**: dados permanecem mesmo após o desligamento ou término da instância EC2
- **Vinculado a uma AZ**: um volume EBS existe dentro de uma única Availability Zone — só pode ser conectado a instâncias da mesma zona
	- Para mover entre zonas: crie um snapshot → restaure o snapshot na zona de destino
- **Um servidor por vez**: por padrão, um EBS só pode ser montado em uma instância ao mesmo tempo
	- A mesma instância pode ter múltiplos volumes (como dois HDs num mesmo computador)
- **Capacidade configurável**: o usuário define o tamanho (GB) e a performance (**IOPS** — número de operações de leitura/escrita por segundo)

---

## Como o EBS Funciona

O EBS **não é um disco físico dentro do servidor**. Ele reside em uma infraestrutura separada da AWS e se conecta à instância EC2 via rede.

**Implicações práticas:**
- A comunicação via rede pode introduzir **latência** em comparação ao armazenamento local
- É possível **desconectar** um volume de uma instância e **reconectar** rapidamente a outra instância na mesma AZ
- A cobrança ocorre pelo volume provisionado, independente do uso

![[Pasted image 20260413103129.png]]

---

## Delete on Termination

Controla o que acontece com o volume EBS quando a instância EC2 associada é **terminada** (não apenas desligada).

![[Pasted image 20260413103156.png]]

| Situação | Comportamento |
|---|---|
| Atributo **ativado** | Volume é deletado automaticamente junto com a EC2 — dados perdidos, cobrança encerrada |
| Atributo **desativado** | Volume persiste após a exclusão da EC2 — pode ser reutilizado, cobrança continua |

### Comportamento padrão

| Volume | Delete on Termination |
|---|---|
| Disco raiz (root) | **Ativado** por padrão |
| Discos adicionais | **Desativado** por padrão |
