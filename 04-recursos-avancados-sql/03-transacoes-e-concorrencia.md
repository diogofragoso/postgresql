# 🐘 Módulo 04: Recursos Avançados SQL
## 📑 Aula 03: Transações, ACID, Níveis de Isolamento e Locks

> **Navegação**: [⬅️ Aula Anterior: Índices e Otimização](./02-indices-e-otimizacao.md) | [Módulo 04](./README.md) | [Próxima Aula: Funções e Procedures ➡️](./04-funcoes-e-stored-procedures.md)

---

### 🎯 Objetivos de Aprendizagem
Ao final desta aula, você será capaz de:
- Controlar transações complexas com `BEGIN`, `COMMIT`, `ROLLBACK` e `SAVEPOINT`.
- Compreender e configurar os **Níveis de Isolamento**: `READ COMMITTED`, `REPEATABLE READ` e `SERIALIZABLE`.
- Conhecer os fenômenos anômalos de concorrência (Leitura Fantasma, Leituras Não-repetíveis).
- Utilizar bloqueios explícitos de linha com **`SELECT FOR UPDATE`** e **`FOR UPDATE SKIP LOCKED`** (padrão de filas de mensageria).
- Identificar e prevenir **Deadlocks**.

---

### 1. Ciclo de Vida de uma Transação e Savepoints

Uma transação agrupa múltiplos comandos SQL em uma única unidade atômica indivisível.

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Aplicação
    participant PG as PostgreSQL
    Dev->>PG: BEGIN;
    Dev->>PG: UPDATE conta SET saldo = saldo - 100 WHERE id = 1;
    Dev->>PG: SAVEPOINT debito_ok;
    Dev->>PG: UPDATE conta SET saldo = saldo + 100 WHERE id = 999; (Erro: ID não existe!)
    Dev->>PG: ROLLBACK TO SAVEPOINT debito_ok;
    Dev->>PG: UPDATE conta SET saldo = saldo + 100 WHERE id = 2; (Conta correta)
    Dev->>PG: COMMIT;
    Note over Dev,PG: Ambas as contas (1 e 2) foram atualizadas de forma segura e atômica.
```

```sql
BEGIN;

-- Primeira etapa
UPDATE contas SET saldo = saldo - 250.00 WHERE id = 10;

-- Criando um ponto de restauração intermediário
SAVEPOINT etapa_1;

-- Tentativa arriscada
INSERT INTO auditoria_log (evento) VALUES ('Tentativa de saque especial');

-- Se algo der errado apenas nesta etapa:
ROLLBACK TO SAVEPOINT etapa_1;

-- Efetivar o restante
COMMIT;
```

---

### 2. Os 4 Níveis de Isolamento de Transação

O padrão ANSI SQL define quatro níveis de isolamento para mitigar anomalias concorrentes. O PostgreSQL **não implementa `READ UNCOMMITTED`**, pois seu mecanismo MVCC garante que leituras sujas (*Dirty Reads*) nunca ocorram!

| Fenômeno Anômalo | `READ COMMITTED` (Padrão) | `REPEATABLE READ` | `SERIALIZABLE` |
| :--- | :---: | :---: | :---: |
| **Dirty Read** (Ler dados não commitados) | ❌ Impossível | ❌ Impossível | ❌ Impossível |
| **Non-repeatable Read** (Mesma query ler dados alterados por outro commit no meio da transação) | ⚠️ Pode Ocorrer | ❌ Impossível | ❌ Impossível |
| **Phantom Read** (Novas linhas inseridas aparecem em contagens repetidas) | ⚠️ Pode Ocorrer | ❌ Impossível | ❌ Impossível |
| **Serialization Anomaly** (Conflitos lógicos complexos entre transações concorrentes) | ⚠️ Pode Ocorrer | ⚠️ Pode Ocorrer | ❌ **Totalmente Impossível** |

```sql
-- Alterando o nível de isolamento da transação atual:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ou o mais estrito:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

> [!NOTE]
> No nível **`SERIALIZABLE`**, se o PostgreSQL detectar um conflito de serialização entre transações concorrentes, ele abortará uma delas com o erro `40001: could not serialize access due to read/write dependencies`. A aplicação backend deve estar preparada para **tentar novamente (retry)** a transação!

---

### 3. Bloqueio Concorrente: `SELECT FOR UPDATE`

Quando dois clientes tentam comprar o último produto em estoque ao mesmo tempo, um `SELECT` comum não impede que ambos leiam `estoque = 1` e tentem subtrair.

Para evitar isso, travamos as linhas selecionadas com **`FOR UPDATE`**:

```sql
BEGIN;

-- Bloqueia a linha daquele produto para outras transações até este commit/rollback:
SELECT estoque 
FROM produtos 
WHERE id = 50 
FOR UPDATE;

-- Outra transação concorrente que tentar rodar FOR UPDATE no id 50 ficará aguardando aqui!

UPDATE produtos 
SET estoque = estoque - 1 
WHERE id = 50;

COMMIT;
```

#### O Padrão Ouro para Filas de Mensageria: `SKIP LOCKED`
Se você tem 10 workers de background processando tarefas pendentes:

```sql
-- Cada worker pega a próxima tarefa livre sem bloquear os outros workers!
SELECT id, payload
FROM fila_de_tarefas
WHERE status = 'pendente'
ORDER BY id ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

---

### 4. Deadlocks: Como Ocorrem e Como Evitar

Um **Deadlock** (Impasse) ocorre quando duas sessões ficam eternamente esperando uma pela outra para liberar uma trava:

```mermaid
sequenceDiagram
    participant S1 as Sessão 1
    participant S2 as Sessão 2
    S1->>S1: Trava Linha A
    S2->>S2: Trava Linha B
    S1->>S2: Tenta travar Linha B (Fica aguardando S2...)
    S2->>S1: Tenta travar Linha A (Fica aguardando S1...)
    Note over S1,S2: DEADLOCK DETECTADO PELO POSTGRESQL!
```

O PostgreSQL possui um detector de deadlock em background (`deadlock_timeout`, padrão 1 segundo). Ao detectar o ciclo, ele aborta automaticamente uma das transações com o erro:
`ERROR: deadlock detected`.

#### Como evitar Deadlocks em 100% das vezes:
> [!TIP]
> **Regra da Ordem Canônica**: Sempre acesse e atualize as tabelas e linhas **na mesma ordem determinística** em todas as partes da sua aplicação (por exemplo, ordenando os IDs de forma crescente: `WHERE id IN (1, 2) ORDER BY id`).

---

### 5. Atividade Prática 05: Laboratório de Concorrência e Simulação de Deadlock

Neste laboratório empolgante, você abrirá duas abas/sessões paralelas (no Beekeeper Studio ou em dois terminais `psql`) para reproduzir intencionalmente um **Deadlock** e ver o detector de ciclos do PostgreSQL entrar em ação!

#### Passo 1: Preparar o Cenário
Execute este script em qualquer uma das sessões para criar os dados:

```sql
CREATE TABLE transferencias_banco (
    conta_id INT PRIMARY KEY,
    titular TEXT NOT NULL,
    saldo NUMERIC(10,2) NOT NULL
);

INSERT INTO transferencias_banco VALUES
    (1, 'Alice', 1000.00),
    (2, 'Bob', 1000.00);
```

#### Passo 2: Executar as Sessões Concorrentes (Abra 2 Abas no Beekeeper)

| Ordem | 🟢 Sessão 1 (Aba 1) | 🔵 Sessão 2 (Aba 2) |
| :---: | :--- | :--- |
| **1** | `BEGIN;` | - |
| **2** | - | `BEGIN;` |
| **3** | `UPDATE transferencias_banco SET saldo = saldo - 100 WHERE conta_id = 1;` *(Trava conta 1!)* | - |
| **4** | - | `UPDATE transferencias_banco SET saldo = saldo - 200 WHERE conta_id = 2;` *(Trava conta 2!)* |
| **5** | `UPDATE transferencias_banco SET saldo = saldo + 100 WHERE conta_id = 2;` *(Fica aguardando a Sessão 2...)* | - |
| **6** | - | `UPDATE transferencias_banco SET saldo = saldo + 200 WHERE conta_id = 1;` *(Dispara o conflito!)* |

<details>
<summary>💡 Clique para entender o que aconteceu e como o Postgres reagiu</summary>

Após cerca de 1 segundo (tempo padrão de `deadlock_timeout`), a Sessão 2 receberá a mensagem:

```text
ERROR: deadlock detected
DETAIL: Process 1420 waits for ShareLock on transaction 892; Process 891 waits for ShareLock on transaction 893.
HINT: See server log for query details.
```

O PostgreSQL abortou a transação da Sessão 2 automaticamente para liberar o travamento da Sessão 1.

**Como consertar no código da aplicação:**
Ordene os IDs antes de executar a query! Ambas as sessões deveriam sempre atualizar a `conta_id = 1` antes da `conta_id = 2`. Assim, a Sessão 2 teria esperado a Sessão 1 terminar a transação inteira antes de iniciar, sem nunca causar um impasse cíclico.
</details>

---

### 📝 Checklist de Transações Seguras

- [ ] Nunca deixo transações abertas esquecidas no backend sem fechar conexão.
- [ ] Conheço o uso de `SAVEPOINT` para tratamento de erros parciais.
- [ ] Uso `FOR UPDATE SKIP LOCKED` para construir sistemas de fila nativos no Postgres.
- [ ] Padronizei a ordem de atualização de entidades para evitar Deadlocks.

---
> **Navegação**: [⬅️ Aula Anterior: Índices e Otimização](./02-indices-e-otimizacao.md) | [Módulo 04](./README.md) | [Próxima Aula: Funções e Procedures ➡️](./04-funcoes-e-stored-procedures.md)
