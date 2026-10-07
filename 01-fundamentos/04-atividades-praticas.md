# 🛠️ Atividades Práticas: Módulo 01 - Fundamentos

<div align="center">

![Nível](https://img.shields.io/badge/Nível-Iniciante_ao_Intermediário-green?style=for-the-badge)
![Tipo](https://img.shields.io/badge/Tipo-Laboratório_Prático-orange?style=for-the-badge)
![Ambiente](https://img.shields.io/badge/Ambiente-Ubuntu_Docker_ou_Neon_Beekeeper-blue?style=for-the-badge)

</div>

Este caderno reúne as atividades práticas do **Módulo 01**. Todas as atividades são organizadas em subitens independentes, permitindo que professores e estudantes pratiquem cada tópico de forma isolada e estruturada.

---

## 📑 Índice de Subitens Práticos

1. [Subitem 1.1: Auditoria de Metadados e Catálogos do PostgreSQL](#-subitem-11-auditoria-de-metadados-e-catalogos-do-postgresql)
2. [Subitem 1.2: Validação de Conectividade com `psql` e Beekeeper Studio Portable](#-subitem-12-validacao-de-conectividade-com-psql-e-beekeeper-studio-portable)
3. [Subitem 1.3: Laboratório de MVCC, Tuplas Mortas e Recuperação com `VACUUM`](#-subitem-13-laboratorio-de-mvcc-tuplas-mortas-e-recuperacao-com-vacuum)

---

## 📌 Subitem 1.1: Auditoria de Metadados e Catálogos do PostgreSQL

### 🎯 Objetivo
Aprender a inspecionar o estado interno do SGBD PostgreSQL consultando os catálogos do sistema (`pg_catalog`), identificando bancos existentes, tabelas de sistema e extensões disponíveis.

### 💻 Ambiente
- **Trilha A**: Terminal do Ubuntu Server (`psql -U postgres -d postgres`) ou container Docker.
- **Trilha B**: Beekeeper Studio Portable ou Neon SQL Editor.

### 📋 Enunciado e Desafio
1. Conecte-se ao seu banco de dados.
2. Escreva uma consulta SQL que liste todos os bancos de dados criados na instância, mostrando seus nomes e respectivos donos (owners).
3. Escreva uma consulta que filtre as tabelas internas do catálogo do sistema cujo nome contenha a palavra `stat`.
4. Descubra quais extensões já estão instaladas no seu banco consultando a view `pg_extension`.
5. Descubra se a extensão `pgcrypto` ou `uuid-ossp` está disponível para instalação no sistema.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 1.1</summary>

```sql
-- 1 e 2. Listar bancos de dados e seus donos
SELECT 
    datname AS nome_banco,
    pg_catalog.pg_get_userbyid(datdba) AS proprietario,
    encoding,
    datcollate AS collate_padrao
FROM pg_database
WHERE datistemplate = false;

-- 3. Filtrar tabelas de estatísticas do sistema
SELECT 
    schemaname AS schema_tabela,
    tablename AS nome_tabela
FROM pg_tables
WHERE tablename LIKE '%stat%'
ORDER BY tablename ASC;

-- 4. Extensões instaladas no banco atual
SELECT 
    extname AS extensao_instalada,
    extversion AS versao
FROM pg_extension;

-- 5. Verificar disponibilidade de extensões no catálogo
SELECT 
    name AS nome_extensao,
    default_version AS versao_padrao,
    comment AS descricao
FROM pg_available_extensions
WHERE name IN ('pgcrypto', 'uuid-ossp', 'vector')
ORDER BY name ASC;
```

**Explicação Pedagógica:**
- O PostgreSQL mantém todos os seus metadados armazenados em tabelas e views regulares dentro do schema `pg_catalog`.
- A função `pg_get_userbyid(datdba)` traduz o identificador numérico interno (OID) do dono para o nome textual do usuário correspondente.
</details>

---

## 📌 Subitem 1.2: Validação de Conectividade com `psql` e Beekeeper Studio Portable

### 🎯 Objetivo
Consolidar a configuração do ambiente de laboratório, testando a conexão via cliente de linha de comando (`psql`) e cliente gráfico portátil (**Beekeeper Studio Portable**).

### 💻 Ambiente
- **Trilha A (Ubuntu Server + Docker)**: CLI Linux e conexão remota TCP/IP na porta `5432`.
- **Trilha B (Neon Cloud)**: Connection String segura via Internet com parâmetro obrigatório `sslmode=require`.

### 📋 Enunciado e Desafio
1. Teste sua conexão executando uma query de handshake que retorne a versão do motor, o nome do banco ativo e a hora exata do servidor.
2. No **Beekeeper Studio Portable**:
   - Abra a aplicação (sem necessidade de instalação de privilégios de administrador).
   - Conecte-se usando a URL de conexão ou os parâmetros manuais (Host, Porta, Usuário, Senha e SSL ativado se Neon).
   - Salve a conexão com o nome `Lab - Aula PostgreSQL`.
3. Escreva um comando para criar uma tabela simples de teste chamada `lab_conexao_teste(id int, mensagem text)`.
4. Insira um registro, consulte-o e, em seguida, remova a tabela com segurança.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 1.2</summary>

```sql
-- 1. Query de handshake e diagnóstico
SELECT 
    version() AS versao_engine,
    current_database() AS banco_conectado,
    current_user AS usuario_sessao,
    CURRENT_TIMESTAMP AS data_hora_servidor;

-- 3. Criação da tabela temporária de validação
CREATE TABLE lab_conexao_teste (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mensagem TEXT NOT NULL,
    criado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 4. Inserção, consulta e limpeza
INSERT INTO lab_conexao_teste (mensagem) 
VALUES ('Conexao com Beekeeper Studio e PostgreSQL validada com sucesso!');

SELECT * FROM lab_conexao_teste;

-- Limpeza de laboratório
DROP TABLE lab_conexao_teste;
```

**Dica de Sala de Aula:**
- Se estiver usando o **Beekeeper Studio Portable**, use o atalho `Ctrl + Enter` (Windows/Linux) para rodar a query selecionada.
- No canto inferior direito da aba de resultados, o Beekeeper exibe a contagem de linhas e o tempo de execução em milissegundos.
</details>

---

## 📌 Subitem 1.3: Laboratório de MVCC, Tuplas Mortas e Recuperação com `VACUUM`

### 🎯 Objetivo
Compreender na prática como o mecanismo de MVCC (*Multi-Version Concurrency Control*) do PostgreSQL lida com atualizações, gerando tuplas mortas (*dead tuples*) e como o comando `VACUUM` atua para recuperar espaço e atualizar ponteiros de visibilidade.

### 💻 Ambiente
- Qualquer instância PostgreSQL 15, 16 ou Neon Cloud.

### 📋 Enunciado e Desafio
1. Crie uma tabela `teste_mvcc (id int primary key, saldo numeric)`.
2. Insira 5 linhas de exemplo com saldos variados.
3. Consulte as colunas ocultas do PostgreSQL (`xmin`, `xmax`, `ctid`) junto com os dados da tabela. Observe o valor de `ctid` (ponteiro físico de bloco e tupla: `(bloco, offset)`).
4. Execute 3 atualizações consecutivas (`UPDATE`) no saldo de uma das linhas.
5. Inspecione novamente o `ctid`, `xmin` e `xmax` da linha alterada. O que aconteceu com o número do bloco/tupla?
6. Inspecione a contagem de tuplas mortas consultando a view `pg_stat_user_tables`.
7. Execute o comando `VACUUM (VERBOSE)` e verifique os logs de liberação.

<details>
<summary>👁️ Clique aqui para ver o script SQL e a explicação do Subitem 1.3</summary>

```sql
-- 1. Criação da tabela de teste
DROP TABLE IF EXISTS teste_mvcc;
CREATE TABLE teste_mvcc (
    id INT PRIMARY KEY,
    saldo NUMERIC(10,2) NOT NULL
);

-- 2. Inserção de dados iniciais
INSERT INTO teste_mvcc (id, saldo) VALUES
    (1, 100.00),
    (2, 250.00),
    (3, 400.00),
    (4, 550.00),
    (5, 700.00);

-- 3. Inspecionando colunas internas de MVCC
SELECT ctid, xmin, xmax, id, saldo FROM teste_mvcc;
-- Observe que a linha com id = 1 tem ctid = (0,1)

-- 4. Executando atualizações consecutivas na mesma linha lógica
UPDATE teste_mvcc SET saldo = saldo + 10 WHERE id = 1;
UPDATE teste_mvcc SET saldo = saldo + 10 WHERE id = 1;
UPDATE teste_mvcc SET saldo = saldo + 10 WHERE id = 1;

-- 5. Consultando novamente as colunas internas
SELECT ctid, xmin, xmax, id, saldo FROM teste_mvcc WHERE id = 1;
-- O ctid mudou! Agora ele aponta para uma nova posição física (ex: (0,6), (0,7) ou (0,8))!

-- 6. Verificando a contagem de tuplas mortas (dead tuples) acumuladas
SELECT 
    relname AS tabela,
    n_live_tup AS tuplas_vivas,
    n_dead_tup AS tuplas_mortas,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'teste_mvcc';

-- 7. Executando o VACUUM manual para marcar o espaço reutilizável
VACUUM (VERBOSE, ANALYZE) teste_mvcc;

-- 8. Limpeza do ambiente
DROP TABLE teste_mvcc;
```

**Explicação Pedagógica:**
- No PostgreSQL, um comando `UPDATE` **nunca sobrescreve os bytes antigos no disco**. Ele marca a tupla anterior como obsoleta (preenchendo seu `xmax` com a transação atual) e grava uma nova tupla em outra página física (`ctid` novo).
- As tuplas antigas continuam ocupando disco como "tuplas mortas" para permitir que transações antigas e isoladas ainda possam lê-las se necessário.
- O `VACUUM` limpa essas referências e adiciona os espaços ao *Free Space Map (FSM)* para reaproveitamento em futuros `INSERTs` e `UPDATEs`.
</details>

---

## 🧭 Navegação
* [⬅️ Voltar para o Módulo 01: Fundamentos](./README.md)
* [Ir para o Módulo 02: Modelagem e DDL ➡️](../02-modelagem-e-ddl/README.md)
