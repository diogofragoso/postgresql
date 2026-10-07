# 🛠️ Atividades Práticas: Módulo 05 - Neon Serverless PostgreSQL

<div align="center">

![Nível](https://img.shields.io/badge/Nível-Cloud_Native_e_DevOps-green?style=for-the-badge)
![Tipo](https://img.shields.io/badge/Tipo-Laboratório_Prático-orange?style=for-the-badge)
![Ambiente](https://img.shields.io/badge/Ambiente-Neon_Cloud_e_Beekeeper_Portable-teal?style=for-the-badge)

</div>

Este caderno reúne as atividades práticas de nuvem serverless do **Módulo 05**. As atividades estão divididas em subitens específicos com exercícios reais de branching, restauração por Point-in-Time Recovery (PITR) e busca semântica de IA com `pgvector`.

---

## 📑 Índice de Subitens Práticos

1. [Subitem 5.1: Conectividade com Direct vs Pooled Endpoints e Beekeeper Studio](#-subitem-51-conectividade-com-direct-vs-pooled-endpoints-e-beekeeper-studio)
2. [Subitem 5.2: Database Branching: Isolamento e Teste de Migração Destrutiva](#-subitem-52-database-branching-isolamento-e-teste-de-migracao-destrutiva)
3. [Subitem 5.3: Simulação de Desastre com TRUNCATE e Resgate Instantâneo com PITR](#-subitem-53-simulacao-de-desastre-com-truncate-e-resgate-instantaneo-com-pitr)
4. [Subitem 5.4: Inteligência Artificial e Busca Semântica Vetorial com HNSW](#-subitem-54-inteligencia-artificial-e-busca-semantica-vetorial-com-hnsw)

---

## 📌 Subitem 5.1: Conectividade com Direct vs Pooled Endpoints e Beekeeper Studio

### 🎯 Objetivo
Entender quando utilizar a conexão direta com o Postgres e quando utilizar o Pooler integrado do Neon (PgBouncer), validando a conexão no **Beekeeper Studio Portable**.

### 💻 Ambiente
- Neon Console e Beekeeper Studio Portable.

### 📋 Enunciado e Desafio
1. Acesse o painel do seu projeto no Neon.
2. Localize a área **Connection Details** no Dashboard.
3. Alterne a caixa de seleção **Connection pooling**:
   - Observe a alteração no domínio do host (inserção do sufixo `-pooler`).
4. Abra o **Beekeeper Studio Portable** e crie duas conexões salvas:
   - Conexão 1: `Neon - Producao Direta` (para migrações de schema DDL).
   - Conexão 2: `Neon - Pooled PgBouncer` (para consultas DML de alta escala).
5. Escreva uma consulta de diagnóstico em ambas as conexões para inspecionar os parâmetros de rede e a versão.

<details>
<summary>👁️ Clique aqui para ver o script SQL e a análise do Subitem 5.1</summary>

```sql
-- Query de validação em qualquer endpoint:
SELECT 
    inet_server_addr() AS ip_remoto_servidor,
    current_database() AS banco_conectado,
    current_user AS usuario_autenticado,
    inet_client_addr() AS ip_origem_estudante,
    version() AS versao_postgresql;
```

**Diretriz Pedagógica:**
- **Direct Endpoint (`ep-xyz.us-east-2.aws.neon.tech`)**: Use para rodar `CREATE TABLE`, `ALTER TABLE`, migrações do Prisma/Liquibase e comandos que exigem recursos que o PgBouncer não suporta em modo transação (como `LISTEN/NOTIFY` ou prepared statements com nomes fixos).
- **Pooled Endpoint (`ep-xyz-pooler.us-east-2.aws.neon.tech`)**: Use em aplicações serverless (Next.js, Vercel, AWS Lambda) onde centenas de funções executam em paralelo sem esgotar o limite de conexões do PostgreSQL.
</details>

---

## 📌 Subitem 5.2: Database Branching: Isolamento e Teste de Migração Destrutiva

### 🎯 Objetivo
Experimentar a funcionalidade de **Database Branching** do Neon, criando um clone instantâneo (*Copy-on-Write*) do banco de produção para testar uma migração destrutiva sem colocar os dados reais em risco.

### 💻 Ambiente
- Neon Console ou CLI `neonctl`.

### 📋 Enunciado e Desafio
1. No branch principal (`main`), crie uma tabela `clientes_vip` com 3 registros.
2. No console do Neon (ou via `neonctl branches create`), crie um novo branch chamado `teste-migracao-destrutiva` com origem no `main`.
3. Conecte-se ao branch `teste-migracao-destrutiva` pelo Beekeeper Studio Portable.
4. Execute uma operação destrutiva: `DROP TABLE clientes_vip CASCADE;`.
5. Volte para a conexão do branch `main` e consulte a tabela `clientes_vip`.
6. Comprove que a tabela de produção continua intacta e com todos os seus dados preservados!

<details>
<summary>👁️ Clique aqui para ver o passo a passo completo do Subitem 5.2</summary>

```sql
-- PASSO 1: No branch 'main' (Produção)
CREATE TABLE clientes_vip (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    nivel_fidelidade VARCHAR(20) DEFAULT 'Diamante'
);

INSERT INTO clientes_vip (nome) VALUES
    ('Arthur Pendelton'),
    ('Beatriz Albuquerque'),
    ('Carlos Drummond');

SELECT * FROM clientes_vip;

-- PASSO 2: Criar branch no Neon Console:
-- Menu lateral -> Branches -> Create Branch
-- Name: teste-migracao-destrutiva | Parent: main

-- PASSO 3 e 4: Conectar ao branch 'teste-migracao-destrutiva' e DESTRUIR:
DROP TABLE clientes_vip CASCADE;

-- Se você der SELECT aqui, receberá:
-- ERROR: relation "clientes_vip" does not exist

-- PASSO 5 e 6: Voltar para a conexão do branch 'main':
SELECT * FROM clientes_vip;
-- As 3 linhas continuam existindo normalmente! O branch isolou a catástrofe!
```

**Explicação Pedagógica:**
- Graças à arquitetura desacoplada do Neon, criar um branch é uma operação de metadados que dura menos de 1 segundo e não consome espaço de armazenamento extra até que você escreva novas páginas modificadas (*Copy-on-Write*).
</details>

---

## 📌 Subitem 5.3: Simulação de Desastre com TRUNCATE e Resgate Instantâneo com PITR

### 🎯 Objetivo
Simular um desastre humano de exclusão acidental de dados em massa com `TRUNCATE` e utilizar o **Point-in-Time Recovery (PITR)** para restaurar os dados com precisão cirúrgica de segundos.

### 💻 Ambiente
- Neon Console.

### 📋 Enunciado e Desafio
1. No branch `main`, crie uma tabela `pedidos_fiscais` e insira 5 notas fiscais.
2. Anote mentalmente o horário exato (ex: 15:42:00) ou consulte `SELECT CLOCK_TIMESTAMP();`.
3. Simule um erro desastroso de um operador: execute `TRUNCATE TABLE pedidos_fiscais;`.
4. No console do Neon, crie um novo branch a partir do `main`, selecionando a opção **"Restore to point in time"** e escolha o horário de 1 minuto antes do desastre.
5. Conecte-se ao novo branch restaurado e confirme que todas as notas fiscais foram salvas sem perda de dados.

<details>
<summary>👁️ Clique aqui para ver o roteiro SQL e operação do Subitem 5.3</summary>

```sql
-- 1. Criação e população
CREATE TABLE pedidos_fiscais (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    chave_nfe CHAR(44) NOT NULL,
    valor_total NUMERIC(10,2) NOT NULL,
    emitido_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO pedidos_fiscais (chave_nfe, valor_total) VALUES
    ('35240100000000000191550010000000011000000010', 1540.00),
    ('35240100000000000191550010000000021000000020', 2890.50),
    ('35240100000000000191550010000000031000000030', 450.00);

-- 2. Registrar o momento anterior ao desastre:
SELECT CLOCK_TIMESTAMP() AS momento_seguro;
-- Exemplo: 2026-10-07 15:45:00-03

-- 3. O DESASTRE (Comando acidental sem backup prévio):
TRUNCATE TABLE pedidos_fiscais;

SELECT COUNT(*) FROM pedidos_fiscais; -- Retorna 0!

-- 4. No Neon Console:
-- Branches -> New Branch -> Time Machine / Point-in-Time
-- Selecione a data e o minuto exato registrado em 'momento_seguro'.
-- Nome do Branch: resgate-pos-desastre

-- 5. Conectando no branch 'resgate-pos-desastre':
SELECT * FROM pedidos_fiscais; -- Todos os registros estão de volta!
```

**Explicação Pedagógica:**
- Os *Safekeepers* e *Pageservers* do Neon mantêm um registro contínuo dos logs de WAL por até 7 dias no plano gratuito. Isso permite que qualquer milissegundo do passado seja reaberto como um banco de dados vivo e funcional.
</details>

---

## 📌 Subitem 5.4: Inteligência Artificial e Busca Semântica Vetorial com HNSW

### 🎯 Objetivo
Habilitar a extensão `vector` no PostgreSQL/Neon, criar embeddings numéricos de teste e aplicar o algoritmo de indexação **HNSW (Hierarchical Navigable Small World)** para buscas semânticas de alta velocidade.

### 💻 Ambiente
- Beekeeper Studio Portable ou Neon SQL Editor.

### 📋 Enunciado e Desafio
1. Ative a extensão vetorial `vector`.
2. Crie a tabela `artigos_base_conhecimento` com `id`, `titulo`, `categoria` e uma coluna `vetor_embedding VECTOR(3)`.
3. Insira 4 artigos representando diferentes temas (ex: programação, culinária e hardware).
4. Crie um índice HNSW usando a métrica de distância do cosseno.
5. Escreva uma busca semântica calculando a distância de cosseno e a porcentagem de proximidade em relação a um conceito de busca.

<details>
<summary>👁️ Clique aqui para ver o script SQL e a solução do Subitem 5.4</summary>

```sql
-- 1. Ativação da extensão de IA
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Tabela de artigos vetoriais
DROP TABLE IF EXISTS artigos_base_conhecimento;
CREATE TABLE artigos_base_conhecimento (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo TEXT NOT NULL,
    categoria TEXT NOT NULL,
    vetor_embedding VECTOR(3) NOT NULL
);

-- 3. Inserção de vetores conceituais normalizados
INSERT INTO artigos_base_conhecimento (titulo, categoria, vetor_embedding) VALUES
    ('Guia de Normalizacao e SQL no PostgreSQL', 'Banco de Dados', '[0.95, 0.05, 0.05]'),
    ('Como preparar sushi e pratos orientais', 'Gastronomia', '[0.05, 0.95, 0.10]'),
    ('Arquitetura de Processadores e Memorias', 'Hardware', '[0.80, 0.10, 0.35]'),
    ('Otimizacao de Queries e Indices no PostgreSQL', 'Banco de Dados', '[0.92, 0.08, 0.04]');

-- 4. Criação do índice HNSW
CREATE INDEX idx_artigos_hnsw 
ON artigos_base_conhecimento 
USING hnsw (vetor_embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 5. Busca semântica: procurando artigos sobre banco de dados ([0.90, 0.05, 0.05])
SELECT 
    titulo,
    categoria,
    ROUND((vetor_embedding <=> '[0.90, 0.05, 0.05]')::NUMERIC, 4) AS distancia_cosseno,
    ROUND(((1 - (vetor_embedding <=> '[0.90, 0.05, 0.05]')) * 100)::NUMERIC, 2) AS similaridade_pct
FROM artigos_base_conhecimento
ORDER BY vetor_embedding <=> '[0.90, 0.05, 0.05]' ASC
LIMIT 2;
```

**Explicação Pedagógica:**
- O operador `<=>` calcula a distância do cosseno entre os dois vetores. Quanto menor a distância, mais conceitualmente próximos os textos estão na semântica da inteligência artificial.
</details>

---

## 🧭 Navegação
* [⬅️ Voltar para o Módulo 05: Neon Cloud](./README.md)
* [Ir para o Módulo 06: Projetos Práticos e Desafios ➡️](../06-projetos-praticos-e-desafios/README.md)
