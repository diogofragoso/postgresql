# 🛠️ Atividades Práticas: Módulo 04 - Recursos Avançados SQL

<div align="center">

![Nível](https://img.shields.io/badge/Nível-Avançado-purple?style=for-the-badge)
![Tipo](https://img.shields.io/badge/Tipo-Laboratório_Prático-orange?style=for-the-badge)
![Ambiente](https://img.shields.io/badge/Ambiente-Ubuntu_Docker_ou_Neon_Beekeeper-blue?style=for-the-badge)

</div>

Este caderno reúne as atividades práticas de engenharia e otimização avançada do **Módulo 04**. Cada atividade é organizada em subitens independentes, permitindo simular cenários de alta concorrência, otimização de consultas e automação com triggers em ambiente de laboratório.

---

## 📑 Índice de Subitens Práticos

1. [Subitem 4.1: Views Virtuais com Segurança e Views Materializadas de Alta Velocidade](#-subitem-41-views-virtuais-com-seguranca-e-views-materializadas-de-alta-velocidade)
2. [Subitem 4.2: Diagnóstico de Performance com `EXPLAIN ANALYZE` e Covering Indexes](#-subitem-42-diagnostico-de-performance-com-explain-analyze-e-covering-indexes)
3. [Subitem 4.3: Laboratório de Concorrência e Simulação de Deadlock em Sessões Paralelas](#-subitem-43-laboratorio-de-concorrencia-e-simulacao-de-deadlock-em-sessoes-paralelas)
4. [Subitem 4.4: Automação com PL/pgSQL: Trigger de Auditoria e Sanitização](#-subitem-44-automacao-com-plpgsql-trigger-de-auditoria-e-sanitizacao)
5. [Subitem 4.5: Análise Temporal com Window Functions e CTEs Recursivas](#-subitem-45-analise-temporal-com-window-functions-e-ctes-recursivas)

---

## 📌 Subitem 4.1: Views Virtuais com Segurança e Views Materializadas de Alta Velocidade

### 🎯 Objetivo
Criar camadas de abstração segura com `VIEW` e acelerar relatórios analíticos pesados usando `MATERIALIZED VIEW` com atualização em segundo plano (`REFRESH MATERIALIZED VIEW CONCURRENTLY`).

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
1. Crie a tabela `vendas_brutas` com 5.000 linhas fictícias contendo data, filial, categoria e valor.
2. Crie uma **View regular** chamada `vw_vendas_seguras` que oculte os identificadores internos e exiba apenas filiais ativas.
3. Crie uma **View Materializada** chamada `mv_resumo_mensal_filiais` que agregue faturamento e contagem por ano, mês e filial.
4. Crie um índice único sobre a View Materializada para permitir atualizações concorrentes sem bloqueio de leitura.
5. Execute o comando `REFRESH MATERIALIZED VIEW CONCURRENTLY` e valide o funcionamento.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 4.1</summary>

```sql
-- 1. Criação da tabela e carga com generate_series
DROP TABLE IF EXISTS vendas_brutas CASCADE;
CREATE TABLE vendas_brutas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    filial VARCHAR(20) NOT NULL,
    categoria VARCHAR(30) NOT NULL,
    valor NUMERIC(10,2) NOT NULL,
    data_venda DATE NOT NULL
);

INSERT INTO vendas_brutas (filial, categoria, valor, data_venda)
SELECT 
    (ARRAY['SP-Capital', 'RJ-Capital', 'MG-BH'])[1 + (random() * 2)::INT],
    (ARRAY['Hardware', 'Perifericos', 'Software'])[1 + (random() * 2)::INT],
    ROUND((random() * 500 + 20)::NUMERIC, 2),
    CURRENT_DATE - (random() * 60)::INT
FROM generate_series(1, 5000);

-- 2. View comum (abstração segura)
CREATE OR REPLACE VIEW vw_vendas_seguras AS
SELECT 
    filial,
    categoria,
    valor,
    data_venda
FROM vendas_brutas
WHERE filial != 'MG-BH'; -- Regra de restrição de acesso

-- 3. View Materializada (persistida em disco)
DROP MATERIALIZED VIEW IF EXISTS mv_resumo_mensal_filiais;
CREATE MATERIALIZED VIEW mv_resumo_mensal_filiais AS
SELECT 
    DATE_TRUNC('month', data_venda)::DATE AS mes_referencia,
    filial,
    COUNT(*) AS total_transacoes,
    SUM(valor) AS faturamento_total
FROM vendas_brutas
GROUP BY DATE_TRUNC('month', data_venda), filial;

-- 4. Índice exclusivo obrigatório para REFRESH CONCURRENTLY
CREATE UNIQUE INDEX idx_mv_resumo_mes_filial 
ON mv_resumo_mensal_filiais (mes_referencia, filial);

-- 5. Atualização não-bloqueante
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_resumo_mensal_filiais;

SELECT * FROM mv_resumo_mensal_filiais ORDER BY mes_referencia DESC, faturamento_total DESC;
```

**Explicação Pedagógica:**
- Sem a cláusula `CONCURRENTLY`, a View Materializada bloqueia leituras (`SELECT`) com um lock exclusivo durante a reconstrução.
- O parâmetro `CONCURRENTLY` exige um `UNIQUE INDEX` na view para rastrear e aplicar deltas em memória sem travar a aplicação.
</details>

---

## 📌 Subitem 4.2: Diagnóstico de Performance com `EXPLAIN ANALYZE` e Covering Indexes

### 🎯 Objetivo
Aprender a ler o plano de execução físico do otimizador de consultas do PostgreSQL, identificar gargalos de *Sequential Scan* em tabelas com alto volume e transformar a query em um **Index Only Scan** utilizando *Covering Index* (`INCLUDE`).

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
1. Crie a tabela `correntistas` com 100.000 registros contendo `cpf`, `agencia`, `saldo` e `ativo`.
2. Execute uma consulta buscando o saldo de correntistas ativos de uma agência específica acompanhada de `EXPLAIN (ANALYZE, BUFFERS)`.
3. Observe o nó gerado (`Seq Scan`) e a quantidade de buffers e milissegundos gastos.
4. Crie um índice B-Tree cobrindo os filtros e incluindo a coluna de retorno (`INCLUDE (saldo)`).
5. Execute a query novamente com `EXPLAIN (ANALYZE, BUFFERS)` e comprove a transição para `Index Only Scan` com custo e tempo próximos de zero.

<details>
<summary>👁️ Clique aqui para ver o script SQL e a análise do Subitem 4.2</summary>

```sql
-- 1. Criação e população de 100.000 linhas
DROP TABLE IF EXISTS correntistas CASCADE;
CREATE TABLE correntistas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cpf CHAR(11) NOT NULL,
    agencia VARCHAR(6) NOT NULL,
    saldo NUMERIC(12,2) NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT true
);

INSERT INTO correntistas (cpf, agencia, saldo, ativo)
SELECT 
    LPAD(s::TEXT, 11, '0'),
    LPAD((1 + (s % 50))::TEXT, 4, '0'),
    ROUND((random() * 10000)::NUMERIC, 2),
    (s % 10 != 0) -- 90% ativos, 10% inativos
FROM generate_series(1, 100000) AS s;

-- 2 e 3. Diagnóstico antes da otimização
EXPLAIN (ANALYZE, BUFFERS)
SELECT saldo
FROM correntistas
WHERE agencia = '0015' AND ativo = true;
-- Observe: Seq Scan on correntistas | Cost elevado | Buffers lidos: centenas de páginas

-- 4. Criação do Covering Index otimizado
CREATE INDEX idx_correntistas_agencia_ativo_covering
ON correntistas (agencia, ativo)
INCLUDE (saldo);

-- 5. Diagnóstico após a criação do índice
EXPLAIN (ANALYZE, BUFFERS)
SELECT saldo
FROM correntistas
WHERE agencia = '0015' AND ativo = true;
-- Observe o resultado: Index Only Scan using idx_correntistas_agencia_ativo_covering!
-- Heap Fetches: 0 (Leitura 100% direta da árvore de índices, sem tocar na tabela principal)
```

**Explicação Pedagógica:**
- No **Index Only Scan**, o PostgreSQL encontra tanto as colunas do filtro (`agencia`, `ativo`) quanto as colunas do SELECT (`saldo`) dentro do próprio arquivo do índice, dispensando o acesso lento à tabela física (*Heap*).
</details>

---

## 📌 Subitem 4.3: Laboratório de Concorrência e Simulação de Deadlock em Sessões Paralelas

### 🎯 Objetivo
Entender o comportamento do gerenciador de travas (*Lock Manager*) do PostgreSQL, reproduzir determinística e intencionalmente um cenário de **Deadlock** utilizando duas abas de conexão independentes e analisar a mensagem de erro emitida pelo motor.

### 💻 Ambiente
- **Duas abas ou conexões abertas no Beekeeper Studio Portable** (ou dois terminais `psql` simultâneos).

### 📋 Enunciado e Desafio
1. Crie a tabela `contas_bancarias (id int primary key, titular text, saldo numeric)`.
2. Insira as contas 1 (Alice) e 2 (Bob) com saldo de R$ 1.000,00 cada.
3. Na **Sessão A**, inicie uma transação e altere o saldo da Conta 1 (bloqueando a linha 1).
4. Na **Sessão B**, inicie outra transação e altere o saldo da Conta 2 (bloqueando a linha 2).
5. Na **Sessão A**, tente alterar a Conta 2 (a Sessão A ficará congelada aguardando a Sessão B).
6. Na **Sessão B**, tente alterar a Conta 1 (cria-se o ciclo mortal de dependência: A espera B e B espera A).
7. Observe o PostgreSQL disparar o *Deadlock Detector* e abortar uma das transações com erro `40P01`.

<details>
<summary>👁️ Clique aqui para ver o passo a passo exato do Subitem 4.3</summary>

```sql
-- Preparação (Execute em qualquer sessão):
DROP TABLE IF EXISTS contas_bancarias CASCADE;
CREATE TABLE contas_bancarias (
    id INT PRIMARY KEY,
    titular VARCHAR(50) NOT NULL,
    saldo NUMERIC(10,2) NOT NULL
);

INSERT INTO contas_bancarias (id, titular, saldo) VALUES
    (1, 'Alice', 1000.00),
    (2, 'Bob', 1000.00);

-- PASSO 1 [Na Aba / Sessão A]:
BEGIN;
UPDATE contas_bancarias SET saldo = saldo - 100 WHERE id = 1;
-- Retorna imediatamente: Linha 1 bloqueada por A.

-- PASSO 2 [Na Aba / Sessão B]:
BEGIN;
UPDATE contas_bancarias SET saldo = saldo - 200 WHERE id = 2;
-- Retorna imediatamente: Linha 2 bloqueada por B.

-- PASSO 3 [Na Aba / Sessão A]:
UPDATE contas_bancarias SET saldo = saldo + 100 WHERE id = 2;
-- A Sessão A trava e fica aguardando B liberar a linha 2!

-- PASSO 4 [Na Aba / Sessão B]:
UPDATE contas_bancarias SET saldo = saldo + 200 WHERE id = 1;
-- O DEADLOCK OCORRE AQUI!
```

**Resultado Emitido pelo PostgreSQL:**
```text
ERROR: deadlock detected
DETAIL: Process 1845 waits for ShareLock on transaction 789; blocked by process 1846.
Process 1846 waits for ShareLock on transaction 788; blocked by process 1845.
HINT: See server log for query details.
```

**Regra de Ouro para Evitar Deadlocks:**
Sempre ordene os recursos de forma determinística antes de atualizar (ex: se uma transação precisa atualizar as contas 1 e 2, garanta no código da aplicação que ela sempre bloqueará a de menor ID primeiro: `ORDER BY id`).
</details>

---

## 📌 Subitem 4.4: Automação com PL/pgSQL: Trigger de Auditoria e Sanitização

### 🎯 Objetivo
Construir gatilhos (*triggers*) em PL/pgSQL para sanitizar dados automaticamente antes da gravação (`BEFORE INSERT OR UPDATE`) e manter uma tabela de log histórico inviolável (`AFTER UPDATE OR DELETE`).

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
1. Crie a tabela `funcionarios (id serial primary key, nome text, cpf text, salario numeric)`.
2. Crie uma tabela `log_auditoria_funcionarios` para registrar: `funcionario_id`, `operacao` ('UPDATE'/'DELETE'), `salario_antigo`, `salario_novo`, `alterado_por` e `data_hora`.
3. Escreva uma função trigger `fn_sanitizar_funcionario` que, antes de inserir ou atualizar, remova espaços extras do nome e remova pontos e traços do CPF (mantendo apenas números).
4. Escreva uma trigger de auditoria que grave no log sempre que o salário for modificado.

<details>
<summary>👁️ Clique aqui para ver o código PL/pgSQL completo do Subitem 4.4</summary>

```sql
-- 1. Tabelas principais e de auditoria
DROP TABLE IF EXISTS log_auditoria_funcionarios CASCADE;
DROP TABLE IF EXISTS funcionarios CASCADE;

CREATE TABLE funcionarios (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    cpf VARCHAR(20) NOT NULL UNIQUE,
    salario NUMERIC(10,2) NOT NULL CHECK (salario > 0)
);

CREATE TABLE log_auditoria_funcionarios (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    funcionario_id INT NOT NULL,
    operacao VARCHAR(10) NOT NULL,
    salario_antigo NUMERIC(10,2),
    salario_novo NUMERIC(10,2),
    alterado_por VARCHAR(50) DEFAULT CURRENT_USER,
    data_hora TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Função de Sanitização (BEFORE)
CREATE OR REPLACE FUNCTION fn_sanitizar_funcionario()
RETURNS TRIGGER AS $$
BEGIN
    NEW.nome := TRIM(NEW.nome);
    NEW.cpf := REGEXP_REPLACE(NEW.cpf, '\D', '', 'g'); -- Remove tudo que não for dígito
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sanitizar_funcionario
BEFORE INSERT OR UPDATE ON funcionarios
FOR EACH ROW EXECUTE FUNCTION fn_sanitizar_funcionario();

-- 3. Função de Auditoria de Salário (AFTER)
CREATE OR REPLACE FUNCTION fn_auditar_salario()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.salario IS DISTINCT FROM NEW.salario THEN
        INSERT INTO log_auditoria_funcionarios (
            funcionario_id, operacao, salario_antigo, salario_novo
        ) VALUES (
            OLD.id, TG_OP, OLD.salario, NEW.salario
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_auditar_salario
AFTER UPDATE ON funcionarios
FOR EACH ROW EXECUTE FUNCTION fn_auditar_salario();

-- 4. Testes práticos
INSERT INTO funcionarios (nome, cpf, salario) 
VALUES ('   Ana Maria Silva   ', '123.456.789-00', 4500.00);

-- Conferindo sanitização automática:
SELECT * FROM funcionarios;

-- Atualizando salário:
UPDATE funcionarios SET salario = 5200.00 WHERE cpf = '12345678900';

-- Conferindo histórico gerado pela trigger:
SELECT * FROM log_auditoria_funcionarios;
```
</details>

---

## 📌 Subitem 4.5: Análise Temporal com Window Functions e CTEs Recursivas

### 🎯 Objetivo
Construir consultas corporativas sofisticadas calculando rankings (`DENSE_RANK`), comparações período a período (`LAG`) e navegando em organogramas de empresas com **Common Table Expressions Recursivas** (`WITH RECURSIVE`).

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
1. Crie uma tabela `colaboradores_hierarquia` com `id`, `nome` e `gestor_id` (que referencia a própria tabela).
2. Insira uma árvore de liderança com Diretor, Gerente, Coordenador e Analista.
3. Escreva uma CTE Recursiva que monte o organograma completo exibindo o nível hierárquico e o caminho do cargo (ex: `Diretoria > Gerência > Coordenação`).
4. Crie uma tabela de faturamento mensal e use a Window Function `LAG` para calcular a taxa de crescimento percentual das vendas de um mês em relação ao mês anterior.

<details>
<summary>👁️ Clique aqui para ver o script SQL e a solução do Subitem 4.5</summary>

```sql
-- 1 e 2. Organograma da empresa
DROP TABLE IF EXISTS colaboradores_hierarquia CASCADE;
CREATE TABLE colaboradores_hierarquia (
    id INT PRIMARY KEY,
    nome VARCHAR(50) NOT NULL,
    cargo VARCHAR(50) NOT NULL,
    gestor_id INT REFERENCES colaboradores_hierarquia(id)
);

INSERT INTO colaboradores_hierarquia (id, nome, cargo, gestor_id) VALUES
    (1, 'Helena Ramos', 'CEO / Diretora', NULL),
    (2, 'Roberto Prado', 'Gerente de Engenharia', 1),
    (3, 'Mariana Costa', 'Gerente Comercial', 1),
    (4, 'Fernando Dias', 'Coordenador de Dados', 2),
    (5, 'Juliana Lins', 'Analista de BI Senior', 4);

-- 3. CTE Recursiva percorrendo a árvore de gestão
WITH RECURSIVE organograma AS (
    -- Âncora: O topo da empresa (gestor_id IS NULL)
    SELECT 
        id,
        nome,
        cargo,
        1 AS nivel_hierarquico,
        cargo::TEXT AS caminho_cargo
    FROM colaboradores_hierarquia
    WHERE gestor_id IS NULL

    UNION ALL

    -- Passo Recursivo: Funcionários vinculados ao nível anterior
    SELECT 
        c.id,
        c.nome,
        c.cargo,
        o.nivel_hierarquico + 1,
        o.caminho_cargo || ' -> ' || c.cargo
    FROM colaboradores_hierarquia c
    JOIN organograma o ON c.gestor_id = o.id
)
SELECT * FROM organograma ORDER BY nivel_hierarquico, id;

-- 4. Análise com Window Function (LAG)
WITH vendas_mensais AS (
    SELECT 1 AS mes, 10000.00 AS total UNION ALL
    SELECT 2 AS mes, 12500.00 AS total UNION ALL
    SELECT 3 AS mes, 11000.00 AS total UNION ALL
    SELECT 4 AS mes, 16500.00 AS total
)
SELECT 
    mes,
    total AS faturamento_atual,
    LAG(total) OVER (ORDER BY mes) AS faturamento_mes_anterior,
    ROUND(
        ((total - LAG(total) OVER (ORDER BY mes)) / LAG(total) OVER (ORDER BY mes)) * 100, 
        2
    ) AS crescimento_percentual
FROM vendas_mensais;
```

**Explicação Pedagógica:**
- `WITH RECURSIVE` executa em duas fases: a consulta âncora inicializa o conjunto de dados, e a consulta recursiva se repete até que nenhum novo registro seja adicionado à árvore.
- A função de janela `LAG(coluna)` acessa valores da linha anterior na partição sem a necessidade de fazer um `JOIN` adicional na mesma tabela.
</details>

---

## 🧭 Navegação
* [⬅️ Voltar para o Módulo 04: Recursos Avançados SQL](./README.md)
* [Ir para o Módulo 05: Neon PostgreSQL Cloud ➡️](../05-neon-postgresql-cloud/README.md)
