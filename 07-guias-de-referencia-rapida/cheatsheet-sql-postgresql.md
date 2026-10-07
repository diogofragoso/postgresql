# 📖 Guia de Referência Rápida
## ⚡ Cheat Sheet: Comandos e Sintaxes Essenciais do PostgreSQL e Neon

> **Navegação**: [⬅️ Módulo 06](../06-projetos-praticos-e-desafios/README.md) | [Módulo 07](./README.md) | [Próximo: Guia de Resolução de Erros ➡️](./guia-resolucao-de-erros.md)

---

### 💻 Meta-Comandos do Cliente `psql`

| Comando | O que faz |
| :--- | :--- |
| `\l` ou `\l+` | Lista todos os bancos de dados (+ tamanho e descrição) |
| `\c nome_banco` | Conecta-se a outro banco de dados |
| `\dt` ou `\dt+` | Lista todas as tabelas do schema atual (+ tamanho) |
| `\d nome_tabela` | Descreve colunas, tipos e constraints da tabela |
| `\d+ nome_tabela`| Descreve detalhes profundos, índices e storage |
| `\di` | Lista todos os índices |
| `\dv` / `\dm` | Lista Views / Views Materializadas |
| `\dn` | Lista todos os Schemas existentes |
| `\du` | Lista todos os usuários e roles de permissão |
| `\x` | Alterna modo de exibição expandido (formato chave: valor vertical) |
| `\timing` | Liga/desliga o cronômetro de tempo de execução |
| `\i arquivo.sql` | Executa comandos de um arquivo SQL externo |
| `\q` | Fecha o psql e volta para o terminal do SO |

---

### 🏗️ DDL: Definição de Estrutura de Tabelas

```sql
-- Criar tabela moderna com identidade e constraints
CREATE TABLE produtos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome TEXT NOT NULL,
    preco NUMERIC(10,2) NOT NULL CHECK (preco > 0),
    categoria_id INT REFERENCES categorias(id) ON DELETE RESTRICT,
    criado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Adicionar coluna com valor padrão seguro
ALTER TABLE produtos ADD COLUMN ativo BOOLEAN DEFAULT TRUE;

-- Adicionar constraint sem travar tabela em produção
ALTER TABLE produtos ADD CONSTRAINT chk_preco_minimo CHECK (preco >= 1.0) NOT VALID;
ALTER TABLE produtos VALIDATE CONSTRAINT chk_preco_minimo;

-- Remover tabela e tudo o que depende dela
DROP TABLE IF EXISTS produtos CASCADE;

-- Limpar tabela resetando o ID
TRUNCATE TABLE produtos RESTART IDENTITY;
```

---

### ✍️ DML: Manipulação e UPSERT

```sql
-- Inserção retornando os dados gerados
INSERT INTO usuarios (nome, email) 
VALUES ('Beatriz', 'beatriz@email.com') 
RETURNING id, criado_em;

-- Padrão UPSERT (Inserir ou Atualizar)
INSERT INTO estoque (produto_id, quantidade)
VALUES (10, 5)
ON CONFLICT (produto_id)
DO UPDATE SET quantidade = estoque.quantidade + EXCLUDED.quantidade;

-- Update seguro com WHERE
UPDATE produtos SET preco = preco * 1.10 WHERE categoria_id = 2;

-- Delete com retorno
DELETE FROM tokens_expirados WHERE expira_em < NOW() RETURNING token;
```

---

### 🔍 Consultas, Filtros e Agregações

```sql
-- Busca case-insensitive e tratamento de nulos
SELECT 
    nome, 
    COALESCE(email_secundario, 'Sem e-mail') AS contato
FROM clientes
WHERE nome ILIKE '%silva%' AND status IN ('ativo', 'pendente');

-- Agrupamento com Cláusula FILTER exclusiva do Postgres
SELECT 
    departamento,
    COUNT(*) AS total_geral,
    COUNT(*) FILTER (WHERE salario > 10000) AS total_altos_salarios,
    ROUND(AVG(salario), 2) AS media_salarial
FROM funcionarios
GROUP BY departamento
HAVING COUNT(*) >= 5;

-- Junção com LATERAL (Top N por grupo)
SELECT c.nome, top_pedidos.id, top_pedidos.total
FROM clientes c
CROSS JOIN LATERAL (
    SELECT id, total FROM pedidos 
    WHERE cliente_id = c.id 
    ORDER BY total DESC LIMIT 1
) AS top_pedidos;
```

---

### 📊 Window Functions e CTEs

```sql
-- CTE Legível
WITH faturamento_por_ano AS (
    SELECT EXTRACT(YEAR FROM data_pedido) AS ano, SUM(total) AS total_ano
    FROM pedidos GROUP BY 1
)
SELECT ano, total_ano FROM faturamento_por_ano ORDER BY ano DESC;

-- Funções de Ranking e Deslocamento
SELECT 
    nome,
    departamento,
    salario,
    DENSE_RANK() OVER (PARTITION BY departamento ORDER BY salario DESC) AS rank_dept,
    LAG(salario, 1) OVER (PARTITION BY departamento ORDER BY salario DESC) AS salario_superior
FROM empregados;
```

---

### ⚡ Neon CLI (`neonctl`) Cheat Sheet

```bash
# Autenticar com a conta do Neon
neonctl auth

# Listar projetos
neonctl projects list

# Criar um branch instantâneo
neonctl branches create --name feature-teste --parent main

# Criar branch apenas com schema (sem dados)
neonctl branches create --name teste-limpo --schema-only

# Criar branch voltando no tempo (PITR)
neonctl branches create --name resgate --parent main --time "2026-10-07T14:00:00Z"

# Obter string de conexão do branch atual
neonctl connection-string --branch feature-teste

# Executar query direta pelo terminal
neonctl sql "SELECT count(*) FROM clientes;"
```

---

### 🐝 Atalhos Rápidos do Beekeeper Studio (Portable)

| Atalho (Windows/Linux) | Atalho (macOS) | Ação no Beekeeper Studio |
| :--- | :--- | :--- |
| `Ctrl + Enter` | `Cmd + Enter` | Executa a query atual onde o cursor está posicionado |
| `Ctrl + Shift + Enter` | `Cmd + Shift + Enter` | Executa todo o conteúdo do editor da aba |
| `Ctrl + Shift + F` | `Cmd + Shift + F` | Formata e indenta o código SQL automaticamente |
| `Ctrl + T` | `Cmd + T` | Abre uma nova aba de editor SQL |
| `Ctrl + W` | `Cmd + W` | Fecha a aba atual |
| `Ctrl + S` | `Cmd + S` | Salva a consulta na lista de consultas salvas |
| `Ctrl + P` | `Cmd + P` | Busca rápida por tabelas ou consultas salvas |
| **Botão "Import from URL"** | **Botão "Import from URL"** | Cola Connection String direta do Docker ou Neon com autoconfiguração |

---
> **Navegação**: [⬅️ Módulo 06](../06-projetos-praticos-e-desafios/README.md) | [Módulo 07](./README.md) | [Próximo: Guia de Resolução de Erros ➡️](./guia-resolucao-de-erros.md)
