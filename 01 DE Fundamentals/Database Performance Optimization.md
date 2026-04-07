# Indexação

Ajuda o banco a encontrar certas linhas mais rápido. Funciona como o índice de um livro.

Sem o índice, o banco pode precisar fazer full table scan, ou seja, ler/varrer a base inteira até encontrar o que foi pedido
Com o índice, ele vai diretamente nos registros mais prováveis

### Exemplo

Imaginar uma tabela de clientes com 50 milhões de linhas

A consulta:
```SQL
SELECT * FROM clientes WHERE cpf = '12345678900';
```

Se `cpf` tiver índice:
- o banco encontra rapidinho

Se não tiver:
- ele pode ler a tabela inteira

![[Pasted image 20260407144113.png]]

# Particionamento

Ajuda o banco ler menos dados. Ele vai dividir uma tabela grande em partes menores, chamadas de partições

Ao invés de uma tabela enorme, ficaria algo como:
`vendas_2023`, `vendas_2024`, `vendas_2025`, etc.

Ou por região, país, etc.

Com isso, se a consulta pede só um pedaço dos dados, o banco lê só aquela partição. Com isso, menos dados são lidos, mais fácil controlar e gerenciar dados antigos e permite processamento em paralelo.

![[Pasted image 20260407144050.png]]

# Compressão

Ajuda o banco a movimentar e armazenar menos dados. Ele reduz o tamanho físico dos dados.