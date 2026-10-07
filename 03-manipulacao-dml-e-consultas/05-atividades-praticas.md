# 🛠️ Atividades Práticas: Módulo 03 - Manipulação DML e Consultas

<div align="center">

![Nível](https://img.shields.io/badge/Nível-Intermediário-blue?style=for-the-badge)
![Tipo](https://img.shields.io/badge/Tipo-Laboratório_Prático-orange?style=for-the-badge)
![Ambiente](https://img.shields.io/badge/Ambiente-Ubuntu_Docker_ou_Neon_Beekeeper-blue?style=for-the-badge)

</div>

Este caderno reúne as atividades práticas do **Módulo 03**. Todas as atividades estão organizadas em subitens práticos independentes, com dados de teste inclusos, enunciados claros e soluções comentadas com tags `<details>`.

---

## 📑 Índice de Subitens Práticos

1. [Subitem 3.1: Operações Atômicas e UPSERT com `ON CONFLICT DO UPDATE`](#-subitem-31-operacoes-atomicas-e-upsert-com-on-conflict-do-update)
2. [Subitem 3.2: Consultas Textuais, Paginação por Cursor e Tratamento de Nulos](#-subitem-32-consultas-textuais-paginacao-por-cursor-e-tratamento-de-nulos)
3. [Subitem 3.3: Agregações Gerenciais e Matrizes de Vendas com `FILTER`](#-subitem-33-agregacoes-gerenciais-e-matrizes-de-vendas-com-filter)
4. [Subitem 3.4: Cruzamento Multitabelas e Detecção de Órfãos com Anti-Joins](#-subitem-34-cruzamento-multitabelas-e-deteccao-de-orfaos-com-anti-joins)

---

## 📌 Subitem 3.1: Operações Atômicas e UPSERT com `ON CONFLICT DO UPDATE`

### 🎯 Objetivo
Praticar comandos `INSERT` em massa (*bulk insert*), retorno atômico de dados com a cláusula `RETURNING` e a técnica de **UPSERT** (*Insert or Update*) sem condições de corrida (*race conditions*).

### 💻 Ambiente
- Beekeeper Studio Portable ou terminal `psql`.

### 📋 Enunciado e Desafio
1. Crie uma tabela `estoque_armazem` contendo:
   - `sku`: Código do produto em texto único (`VARCHAR(20) PRIMARY KEY`).
   - `nome_produto`: Nome descritivo (`VARCHAR(100) NOT NULL`).
   - `quantidade_disponivel`: Quantidade inteira maior ou igual a zero.
   - `ultima_atualizacao`: `TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP`.
2. Insira em uma única instrução `INSERT` 3 produtos iniciais usando `RETURNING sku, quantidade_disponivel`.
3. Escreva um script de integração simulando a chegada de uma remessa de caminhão com os produtos:
   - Se o SKU já existir, adicione a nova quantidade ao estoque existente e atualize a data de movimentação.
   - Se o SKU não existir, cadastre o novo produto.
4. Use a cláusula `RETURNING` para conferir o saldo final resultante de cada item após a operação.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 3.1</summary>

```sql
-- 1. Criação da tabela
DROP TABLE IF EXISTS estoque_armazem;
CREATE TABLE estoque_armazem (
    sku VARCHAR(20) PRIMARY KEY,
    nome_produto VARCHAR(100) NOT NULL,
    quantidade_disponivel INT NOT NULL CHECK (quantidade_disponivel >= 0),
    ultima_atualizacao TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Carga inicial em lote com RETURNING
INSERT INTO estoque_armazem (sku, nome_produto, quantidade_disponivel) VALUES
    ('TEC-KB-01', 'Teclado Mecanico RGB', 15),
    ('TEC-MS-02', 'Mouse Optico 16000 DPI', 25),
    ('MON-27-03', 'Monitor 27 Pol 144Hz', 8)
RETURNING sku, nome_produto, quantidade_disponivel;

-- 3 e 4. Carga de remessa com UPSERT e RETURNING
INSERT INTO estoque_armazem (sku, nome_produto, quantidade_disponivel) VALUES
    ('TEC-KB-01', 'Teclado Mecanico RGB', 10),    -- Já existe: Deve somar +10 (total 25)
    ('HD-EXT-04', 'SSD Externo 1TB USB-C', 30)     -- Novo: Deve ser inserido
ON CONFLICT (sku) 
DO UPDATE SET
    quantidade_disponivel = estoque_armazem.quantidade_disponivel + EXCLUDED.quantidade_disponivel,
    ultima_atualizacao = CURRENT_TIMESTAMP
RETURNING sku, quantidade_disponivel AS quantidade_final, ultima_atualizacao;
```

**Explicação Pedagógica:**
- A pseudo-tabela `EXCLUDED` armazena os valores que seriam inseridos caso não houvesse colisão na chave única (`sku`).
- Essa operação é 100% atômica no motor do PostgreSQL, evitando inconsistências causadas por verificações manuais de `SELECT` seguidas de `INSERT` ou `UPDATE`.
</details>

---

## 📌 Subitem 3.2: Consultas Textuais, Paginação por Cursor e Tratamento de Nulos

### 🎯 Objetivo
Dominar buscas textuais insensíveis a maiúsculas/minúsculas (`ILIKE`), técnicas de paginação estável de alta performance (Cursor Pagination com `WHERE id > :ultimo_id`) e funções de tratamento de nulos (`COALESCE`).

### 💻 Ambiente
- Beekeeper Studio Portable ou Neon Cloud.

### 📋 Enunciado e Desafio
1. Crie uma tabela `clientes_crm` com `id`, `nome`, `sobrenome`, `telefone` (que pode ser nulo) e `cidade`.
2. Insira 5 clientes, deixando alguns sem telefone cadastrado.
3. Escreva uma consulta que retorne:
   - O nome completo concatenado (`nome || ' ' || sobrenome`).
   - O telefone formatado ou o texto `'Telefone não informado'` caso seja nulo (usando `COALESCE`).
   - Filtrar apenas clientes cujo nome ou sobrenome contenha `'silva'` (sem diferenciar maiúsculas de minúsculas).
4. Implemente uma consulta que demonstre o conceito de **paginação por cursor (keyset pagination)**, trazendo a próxima página de 2 registros a partir do `id = 2`.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 3.2</summary>

```sql
-- 1. Criação da tabela
DROP TABLE IF EXISTS clientes_crm;
CREATE TABLE clientes_crm (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome VARCHAR(50) NOT NULL,
    sobrenome VARCHAR(50) NOT NULL,
    telefone VARCHAR(20),
    cidade VARCHAR(50) NOT NULL
);

-- 2. Inserção de dados
INSERT INTO clientes_crm (nome, sobrenome, telefone, cidade) VALUES
    ('Carlos', 'Silva', '11-98765-4321', 'Sao Paulo'),
    ('Beatriz', 'Oliveira', NULL, 'Campinas'),
    ('Lucas', 'da Silva', NULL, 'Rio de Janeiro'),
    ('Mariana', 'Souza', '21-99999-8888', 'Niteroi'),
    ('Ricardo', 'Silva Santos', '31-97777-6666', 'Belo Horizonte');

-- 3. Consulta com concatenação, COALESCE e ILIKE
SELECT 
    id,
    nome || ' ' || sobrenome AS nome_completo,
    COALESCE(telefone, 'Telefone não informado') AS contato,
    cidade
FROM clientes_crm
WHERE (nome ILIKE '%silva%' OR sobrenome ILIKE '%silva%')
ORDER BY id ASC;

-- 4. Paginação estável por cursor (próxima página após id 2, limite 2)
SELECT 
    id,
    nome || ' ' || sobrenome AS nome_completo,
    cidade
FROM clientes_crm
WHERE id > 2
ORDER BY id ASC
LIMIT 2;
```

**Explicação Pedagógica:**
- `ILIKE` é específico do PostgreSQL e faz correspondência textual insensível a caixa alta/baixa.
- A paginação por cursor (`WHERE id > :ultimo_id ORDER BY id LIMIT N`) é centenas de vezes mais rápida que `OFFSET`, pois aproveita o índice da chave primária sem descartar leituras intermediárias na memória.
</details>

---

## 📌 Subitem 3.3: Agregações Gerenciais e Matrizes de Vendas com `FILTER`

### 🎯 Objetivo
Construir relatórios analíticos utilizando funções de agregação (`COUNT`, `SUM`, `AVG`, `MAX`), agrupamentos com `GROUP BY`, regras de pós-filtragem com `HAVING` e a cláusula moderna `FILTER (WHERE ...)`.

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
1. Crie a tabela `faturamento_lojas` com colunas: `loja_regiao` (ex: Sul, Sudeste, Nordeste), `categoria_produto`, `status_pagamento` ('aprovado' ou 'cancelado') e `valor_total`.
2. Insira 8 transações de teste.
3. Escreva uma consulta gerencial que calcule, para cada região:
   - O total geral de transações.
   - O faturamento somado apenas de pedidos aprovados (usando a cláusula `FILTER (WHERE status_pagamento = 'aprovado')`).
   - O ticket médio de vendas aprovadas formatado com duas casas decimais.
4. Adicione uma cláusula `HAVING` para exibir apenas regiões que faturaram mais de R$ 500,00 no total de vendas aprovadas.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 3.3</summary>

```sql
-- 1. Criação da tabela
DROP TABLE IF EXISTS faturamento_lojas;
CREATE TABLE faturamento_lojas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    loja_regiao VARCHAR(30) NOT NULL,
    categoria_produto VARCHAR(50) NOT NULL,
    status_pagamento VARCHAR(20) NOT NULL,
    valor_total NUMERIC(10,2) NOT NULL
);

-- 2. Inserção de dados
INSERT INTO faturamento_lojas (loja_regiao, categoria_produto, status_pagamento, valor_total) VALUES
    ('Sudeste', 'Informatica', 'aprovado', 1200.00),
    ('Sudeste', 'Smartphones', 'cancelado', 3500.00),
    ('Sudeste', 'Acessorios', 'aprovado', 150.00),
    ('Sul', 'Informatica', 'aprovado', 850.00),
    ('Sul', 'Smartphones', 'aprovado', 2200.00),
    ('Nordeste', 'Acessorios', 'aprovado', 80.00),
    ('Nordeste', 'Acessorios', 'cancelado', 120.00),
    ('Nordeste', 'Informatica', 'aprovado', 250.00);

-- 3 e 4. Relatório analítico com FILTER e HAVING
SELECT 
    loja_regiao,
    COUNT(*) AS total_pedidos_geral,
    COUNT(*) FILTER (WHERE status_pagamento = 'aprovado') AS qtd_aprovados,
    COUNT(*) FILTER (WHERE status_pagamento = 'cancelado') AS qtd_cancelados,
    SUM(valor_total) FILTER (WHERE status_pagamento = 'aprovado') AS faturamento_aprovado,
    ROUND(AVG(valor_total) FILTER (WHERE status_pagamento = 'aprovado'), 2) AS ticket_medio_aprovado
FROM faturamento_lojas
GROUP BY loja_regiao
HAVING COALESCE(SUM(valor_total) FILTER (WHERE status_pagamento = 'aprovado'), 0) > 500.00
ORDER BY faturamento_aprovado DESC;
```

**Explicação Pedagógica:**
- A cláusula `FILTER (WHERE ...)` é um recurso do padrão SQL:2003 nativo e otimizado no PostgreSQL. Ela substitui com extrema clareza sintática construções arcaicas como `SUM(CASE WHEN status = 'aprovado' THEN valor ELSE 0 END)`.
</details>

---

## 📌 Subitem 3.4: Cruzamento Multitabelas e Detecção de Órfãos com Anti-Joins

### 🎯 Objetivo
Praticar junções relacionais entre tabelas (`INNER JOIN`, `LEFT JOIN`, `FULL OUTER JOIN`) e dominar o padrão de **Anti-Join** para encontrar registros órfãos ou clientes inativos.

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
1. Crie duas tabelas:
   - `turmas (id serial primary key, nome_turma varchar(50))`
   - `estudantes (id serial primary key, turma_id int references turmas(id), nome_aluno varchar(100))`
2. Insira dados de forma que:
   - Haja uma turma com estudantes vinculados.
   - Haja uma turma vazia (sem nenhum estudante).
   - Haja um estudante sem turma (`turma_id` é nulo).
3. Escreva queries para:
   - Listar todas as turmas e a contagem de alunos matriculados (usando `LEFT JOIN`, garantindo que turmas vazias mostrem `0`).
   - Identificar quais turmas **não possuem nenhum aluno matriculado** (padrão Anti-Join com `LEFT JOIN ... WHERE ... IS NULL`).
   - Listar todos os alunos e turmas, incluindo tanto alunos sem turma quanto turmas sem alunos (usando `FULL OUTER JOIN`).

<details>
<summary>👁️ Clique aqui para ver o script SQL e a explicação do Subitem 3.4</summary>

```sql
-- 1. Criação das tabelas
DROP TABLE IF EXISTS estudantes CASCADE;
DROP TABLE IF EXISTS turmas CASCADE;

CREATE TABLE turmas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome_turma VARCHAR(50) NOT NULL
);

CREATE TABLE estudantes (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    turma_id INT REFERENCES turmas(id) ON DELETE SET NULL,
    nome_aluno VARCHAR(100) NOT NULL
);

-- 2. Inserção dos cenários
INSERT INTO turmas (nome_turma) VALUES
    ('Turma Banco de Dados A'),
    ('Turma Redes e Infra B'),
    ('Turma Inteligencia Artificial C (Vazia)');

INSERT INTO estudantes (turma_id, nome_aluno) VALUES
    (1, 'Alice Santos'),
    (1, 'Bruno Mendes'),
    (2, 'Carla Ferreira'),
    (NULL, 'Daniel Sem Turma');

-- 3A. Listar todas as turmas e contagem de alunos (com LEFT JOIN)
SELECT 
    t.nome_turma,
    COUNT(e.id) AS total_matriculados
FROM turmas t
LEFT JOIN estudantes e ON t.id = e.turma_id
GROUP BY t.id, t.nome_turma
ORDER BY total_matriculados DESC;

-- 3B. Anti-Join: Descobrir turmas sem nenhum aluno
SELECT 
    t.id AS turma_id,
    t.nome_turma
FROM turmas t
LEFT JOIN estudantes e ON t.id = e.turma_id
WHERE e.id IS NULL;

-- 3C. FULL OUTER JOIN: Cruzamento bidirecional completo
SELECT 
    COALESCE(t.nome_turma, 'Sem Turma Atribuida') AS turma,
    COALESCE(e.nome_aluno, 'Nenhum Aluno Matriculado') AS estudante
FROM turmas t
FULL OUTER JOIN estudantes e ON t.id = e.turma_id
ORDER BY t.nome_turma NULLS LAST;
```

**Explicação Pedagógica:**
- No item 3A, utilizamos `COUNT(e.id)` em vez de `COUNT(*)`. O `COUNT(*)` contaria a linha vazia gerada pelo `LEFT JOIN` e retornaria incorretamente `1` para a turma vazia, enquanto `COUNT(coluna)` ignora valores `NULL`.
</details>

---

## 🧭 Navegação
* [⬅️ Voltar para o Módulo 03: Manipulação DML](./README.md)
* [Ir para o Módulo 04: Recursos Avançados SQL ➡️](../04-recursos-avancados-sql/README.md)
