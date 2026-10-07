# 📖 Guia de Referência Rápida
## 🛠️ Guia de Resolução de Erros Comuns (Troubleshooting)

> **Navegação**: [⬅️ Cheat Sheet](./cheatsheet-sql-postgresql.md) | [Módulo 07](./README.md) | [🏠 Início](../README.md)

---

Este guia compila os 10 erros mais frequentes que estudantes e desenvolvedores encontram ao trabalhar com PostgreSQL e Neon, com suas causas raízes e soluções passo a passo.

---

### 1. `Connection refused (port 5432)`
```text
psql: error: connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused
Is the server running on that host and accepting TCP/IP connections?
```
* **Causa**: O serviço do PostgreSQL local ou container Docker não está em execução, ou está escutando em outra porta.
* **Como Resolver**:
  1. Se estiver usando Docker, verifique se o container está ativo:
     ```bash
     docker ps
     # Se não estiver rodando:
     docker compose up -d
     ```
  2. Verifique se a porta 5432 não está bloqueada por outro processo.

---

### 2. `Password authentication failed for user`
```text
FATAL: password authentication failed for user "admin"
```
* **Causa**: Senha incorreta ou usuário inexistente no arquivo de permissões (`pg_hba.conf`).
* **Como Resolver**:
  1. Verifique a variável `POSTGRES_PASSWORD` configurada no seu `compose.yaml`.
  2. No Neon, certifique-se de que não copiou espaços invisíveis antes ou depois da senha na Connection String.

---

### 3. `relation "..." does not exist`
```text
ERROR: relation "clientes" does not exist
LINE 1: SELECT * FROM clientes;
```
* **Causa**: 
  - A tabela foi criada dentro de outro schema (ex.: `academico.clientes`) e você não especificou o prefixo nem o `search_path`.
  - A tabela foi criada com aspas duplas e letras maiúsculas (`CREATE TABLE "Clientes"`), tornando o identificador estritamente sensível a maiúsculas.
* **Como Resolver**:
  1. Verifique se o schema está correto: `SELECT * FROM meu_schema.clientes;`.
  2. Ajuste o search path: `SET search_path TO meu_schema, public;`.
  3. No PostgreSQL, **sempre crie tabelas e colunas em minúsculas e snake_case** para evitar esse problema.

---

### 4. `column "..." does not exist` (Aspas Simples vs Aspas Duplas)
```text
ERROR: column "ativo" does not exist
LINE 1: SELECT * FROM produtos WHERE status = "ativo";
```
* **Causa**: Confusão clássica entre tipos de aspas!
  - **Aspas simples (`'texto'`)**: Delimitam **valores de strings/texto**.
  - **Aspas duplas (`"coluna"`)**: Delimitam **nomes de identificadores (tabelas e colunas)**.
* **Como Resolver**:
  ```sql
  -- Errado:
  SELECT * FROM produtos WHERE status = "ativo";

  -- Correto:
  SELECT * FROM produtos WHERE status = 'ativo';
  ```

---

### 5. `duplicate key value violates unique constraint`
```text
ERROR: duplicate key value violates unique constraint "clientes_email_key"
DETAIL: Key (email)=(ana@email.com) already exists.
```
* **Causa**: Tentativa de inserir um registro cujo valor em coluna com restrição `UNIQUE` ou `PRIMARY KEY` já existe no banco.
* **Como Resolver**:
  - Trate a situação usando o padrão **UPSERT**:
    ```sql
    INSERT INTO clientes (nome, email) VALUES ('Ana', 'ana@email.com')
    ON CONFLICT (email) DO NOTHING; -- ou DO UPDATE SET ...
    ```

---

### 6. `violates foreign key constraint`
```text
ERROR: update or delete on table "categorias" violates foreign key constraint "produtos_categoria_id_fkey" on table "produtos"
DETAIL: Key (id)=(1) is still referenced from table "produtos".
```
* **Causa**: Você tentou excluir uma linha mãe que possui registros filhos dependentes dela.
* **Como Resolver**:
  1. Remova ou atualize primeiro os registros filhos na tabela `produtos`.
  2. Se a intenção for realmente apagar tudo em conjunto, configure a constraint com `ON DELETE CASCADE` na criação da tabela.

---

### 7. `must appear in the GROUP BY clause or be used in an aggregate function`
```text
ERROR: column "produtos.nome" must appear in the GROUP BY clause or be used in an aggregate function
```
* **Causa**: Você adicionou uma coluna no `SELECT` sem colocá-la no `GROUP BY` e sem envolvê-la em uma função como `SUM`, `AVG` ou `MAX`.
* **Como Resolver**:
  - Adicione a coluna faltante na cláusula `GROUP BY`:
    ```sql
    -- Errado:
    SELECT categoria_id, nome, SUM(preco) FROM produtos GROUP BY categoria_id;

    -- Correto:
    SELECT categoria_id, nome, SUM(preco) FROM produtos GROUP BY categoria_id, nome;
    ```

---

### 8. `SSL connection error` ou timeout no Neon
```text
FATAL: no pg_hba.conf entry for host "...", user "...", database "...", no encryption
```
* **Causa**: O Neon exige estritamente criptografia TLS/SSL na conexão externa.
* **Como Resolver**:
  - Adicione o parâmetro obrigatório ao final da string de conexão:
    ```text
    ?sslmode=require
    ```

---

### 9. Demora de 1 a 2 segundos na primeira query no Neon (Cold Start)
* **Causa**: A computação estava em modo **Scale to Zero** (suspensa por inatividade) para economizar recursos.
* **Como Resolver**:
  - Esse comportamento é o funcionamento normal e esperado da arquitetura serverless. Após acordar, todas as queries seguintes responderão em milissegundos.
  - Para ambientes de produção com tráfego ininterrupto, é possível configurar o tempo de inatividade para um valor maior no painel do Neon.

---

### 10. `deadlock detected`
```text
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 678; Process 678 waits for ShareLock on transaction 12345.
```
* **Causa**: Duas sessões concorrentes bloquearam recursos diferentes e estão tentando acessar o recurso uma da outra em ordem cruzada.
* **Como Resolver**:
  1. Implemente **tentativa automática (*retry*)** no código backend ao receber o erro `40P01`.
  2. Padronize a ordem de atualização das tabelas e linhas em todas as transações da sua aplicação.

---
> **Navegação**: [⬅️ Cheat Sheet](./cheatsheet-sql-postgresql.md) | [Módulo 07](./README.md) | [🏠 Início](../README.md)
