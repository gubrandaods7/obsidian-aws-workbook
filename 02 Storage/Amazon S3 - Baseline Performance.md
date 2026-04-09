
## Índice

- [[#Upload Multipart (Multi-Part upload):]]
- [[#Aceleração de Transferência do S3 (S3 Transfer Acceleration):]]

---

- O Amazon S3 escala automaticamente para altas taxas de requisição, com latência de 100–200ms

- Sua aplicação pode atingir pelo menos 3.500 requisições PUT/COPY/POST/DELETE ou 5.500 requisições GET/HEAD por segundo por prefixo em um bucket

- Não há limites para o número de prefixos em um bucket

- Exemplo (caminho do objeto ->  prefixo):
    - bucket/folder1/sub1/file ->  `/folder1/sub1/`
    - bucket/folder1/sub2/file ->  `/folder1/sub2/`
    - bucket/1/file ->  `/1/`
    - bucket/2/file ->  `/2/`

- Se você distribuir as leituras igualmente entre os quatro prefixos, pode alcançar 22.000 requisições por segundo para GET e HEAD

## Upload Multipart (Multi-Part upload):

- Recomendado para arquivos > 100 MB, obrigatório para arquivos > 5 GB
- Pode ajudar a paralelizar uploads (aumentar a velocidade de transferência)

![[Pasted image 20260408155012.png|367]]

## Aceleração de Transferência do S3 (S3 Transfer Acceleration):

- Aumenta a velocidade de transferência enviando o arquivo para uma edge location da AWS, que encaminha os dados para o bucket S3 na região de destino
- Compatível com upload multiparte

![[Pasted image 20260408155032.png|464]]

