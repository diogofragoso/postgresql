# 🚀 Módulo 06: Projetos Práticos e Desafios
## 📝 Banco de 30 Questões Práticas com Gabarito Retrátil

> **Navegação**: [⬅️ Aula Anterior: Projeto E-Commerce](./01-projeto-ecommerce.md) | [Módulo 06](./README.md) | [Módulo 07: Guias de Referência ➡️](../07-guias-de-referencia-rapida/README.md)

---

Este banco de exercícios foi desenvolvido para fixação individual e avaliações práticas de alunos. Tente resolver cada exercício em seu banco antes de abrir a aba com a resposta.

---

### 🟢 Nível 1: Fundamentos, Tipos e DDL (Questões 01 a 10)

#### Questão 01: Criação de Tabela com Chave Primária Moderna
Crie uma tabela `fornecedores` com:
- `id` inteiro auto-incremental usando a sintaxe moderna SQL:2003 `GENERATED ALWAYS AS IDENTITY`.
- `razao_social` em texto obrigatório.
- `cnpj` com exatamente 14 caracteres numéricos e restrição de unicidade.
- `ativo` booleano com valor padrão `TRUE`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE fornecedores (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    razao_social TEXT NOT NULL,
    cnpj VARCHAR(14) NOT NULL UNIQUE,
    ativo BOOLEAN NOT NULL DEFAULT TRUE,
    CONSTRAINT chk_cnpj_tamanho CHECK (length(cnpj) = 14)
);
```
</details>

---

#### Questão 02: Check Constraint de Idade Mínima
Crie uma tabela `motoristas` onde a data de nascimento (`data_nascimento`) impeça o cadastro de motoristas com menos de 18 anos completos na data do registro.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE motoristas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome TEXT NOT NULL,
    cnh VARCHAR(11) NOT NULL UNIQUE,
    data_nascimento DATE NOT NULL,
    CONSTRAINT chk_motorista_maior_idade 
        CHECK (data_nascimento <= CURRENT_DATE - INTERVAL '18 years')
);
```
</details>

---

#### Questão 03: Tabela Associativa com Chave Composta
Crie as tabelas `alunos` e `cursos` e uma tabela associativa `inscricoes` representando um relacionamento N:N, onde a chave primária seja composta pela combinação do `aluno_id` e `curso_id`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE alunos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome TEXT NOT NULL
);

CREATE TABLE cursos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo TEXT NOT NULL
);

CREATE TABLE inscricoes (
    aluno_id INT REFERENCES alunos(id) ON DELETE CASCADE,
    curso_id INT REFERENCES cursos(id) ON DELETE CASCADE,
    data_inscricao DATE DEFAULT CURRENT_DATE,
    PRIMARY KEY (aluno_id, curso_id)
);
```
</details>

---

#### Questão 04: Tipo Enumerado Customizado
Crie um tipo enumerado `prioridade_chamado` com os valores `'baixa'`, `'media'`, `'alta'`, `'critica'` e uma tabela `chamados_suporte` que utilize esse tipo com valor padrão `'media'`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TYPE prioridade_chamado AS ENUM ('baixa', 'media', 'alta', 'critica');

CREATE TABLE chamados_suporte (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    assunto TEXT NOT NULL,
    prioridade prioridade_chamado DEFAULT 'media',
    aberto_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```
</details>

---

#### Questão 05: Coluna Gerada (Stored Generated Column)
Crie uma tabela `retangulos` com colunas `largura` e `altura` (ambas `NUMERIC(10,2)`) e uma coluna `area` que calcule e armazene automaticamente a multiplicação de largura por altura.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE retangulos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    largura NUMERIC(10,2) NOT NULL CHECK (largura > 0),
    altura NUMERIC(10,2) NOT NULL CHECK (altura > 0),
    area NUMERIC(10,2) GENERATED ALWAYS AS (largura * altura) STORED
);
```
</details>

---

#### Questão 06: UUID Nativo
Crie uma tabela `sessoes_web` onde a chave primária seja do tipo `UUID` gerada automaticamente pela função nativa `gen_random_uuid()`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE sessoes_web (
    token UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usuario_id INT NOT NULL,
    ip_origem INET NOT NULL,
    expira_em TIMESTAMPTZ NOT NULL
);
```
</details>

---

#### Questão 07: Adicionando Coluna em Produção com NOT VALID
Escreva o comando DDL para adicionar uma restrição de verificação `chk_saldo_positivo` na tabela `contas`, garantindo que ela não bloqueie leituras e escritas concorrentes na criação.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
-- Passo 1: Adiciona instantaneamente validando apenas novas linhas
ALTER TABLE contas 
    ADD CONSTRAINT chk_saldo_positivo CHECK (saldo >= 0) NOT VALID;

-- Passo 2: Valida os dados existentes em background sem travar escritas
ALTER TABLE contas 
    VALIDATE CONSTRAINT chk_saldo_positivo;
```
</details>

---

#### Questão 08: Chave Estrangeira com ON DELETE SET NULL
Explique e demonstre a sintaxe de uma chave estrangeira onde, se o usuário pai for deletado, a coluna de referência nos posts do blog fique como `NULL` em vez de apagar o artigo.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE artigos_blog (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo TEXT NOT NULL,
    autor_id INT REFERENCES usuarios(id) ON DELETE SET NULL
);
```
*Se o autor com ID 5 for removido, o artigo permanecerá intacto na tabela, e sua coluna `autor_id` passará a valer `NULL`.*
</details>

---

#### Questão 09: Manipulação de Schemas
Crie um schema chamado `auditoria` e configure a sessão para procurar tabelas prioritariamente no schema `auditoria` antes do `public`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE SCHEMA IF NOT EXISTS auditoria;

SET search_path TO auditoria, public;
```
</details>

---

#### Questão 10: Limpeza Total vs Preservação de Estrutura
Qual comando esvazia imediatamente todas as 5 milhões de linhas de uma tabela `logs_acesso`, redefinindo os contadores de auto-incremento para 1 sem excluir a tabela?

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
TRUNCATE TABLE logs_acesso RESTART IDENTITY;
```
</details>

---

### 🟡 Nível 2: DML, Consultas, Filtros e Agregações (Questões 11 a 20)

#### Questão 11: Inserção com Cláusula RETURNING
Insira um novo registro na tabela `clientes` informando apenas `nome` e `email`, retornando imediatamente o `id` gerado e a data `criado_em`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
INSERT INTO clientes (nome, email, cpf, endereco)
VALUES ('Marcos Paulo', 'marcos@teste.com', '99988877766', '{"cidade": "Campinas"}'::jsonb)
RETURNING id, criado_em;
```
</details>

---

#### Questão 12: Padrão UPSERT com EXCLUDED
Escreva um comando de inserção na tabela `produtos` (chave única: `sku`). Se o SKU já existir, atualize o `preco` para o novo preço enviado e some o novo `estoque` ao existente.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
INSERT INTO produtos (sku, nome, preco, estoque, categoria_id)
VALUES ('MOUSE-GAMER-RGB', 'Mouse Gamer Pro', 199.90, 15, 2)
ON CONFLICT (sku) 
DO UPDATE SET
    preco = EXCLUDED.preco,
    estoque = produtos.estoque + EXCLUDED.estoque;
```
</details>

---

#### Questão 13: Busca Case-Insensitive com ILIKE
Selecione todos os clientes cujo nome contenha o termo `"silva"`, independentemente de estar grafado com maiúsculas, minúsculas ou misto.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT * FROM clientes
WHERE nome ILIKE '%silva%';
```
</details>

---

#### Questão 14: Tratamento de Nulos com COALESCE
Construa uma consulta que selecione o nome do cliente e seu telefone comercial. Se o telefone comercial for nulo, selecione o celular; se o celular também for nulo, exiba a mensagem `'Nenhum telefone cadastrado'`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    nome,
    COALESCE(telefone_comercial, telefone_celular, 'Nenhum telefone cadastrado') AS contato_principal
FROM contatos;
```
</details>

---

#### Questão 15: Filtragem Agregada com HAVING
Escreva uma consulta que retorne as categorias de produtos que possuem mais de 10 produtos cadastrados com preço médio superior a R$ 100,00.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    categoria_id,
    COUNT(*) AS total_produtos,
    ROUND(AVG(preco), 2) AS preco_medio
FROM produtos
GROUP BY categoria_id
HAVING COUNT(*) > 10 AND AVG(preco) > 100.00;
```
</details>

---

#### Questão 16: Cláusula FILTER em Agregação
Em uma única consulta na tabela `pedidos`, mostre o faturamento total, o faturamento exclusivo de pedidos entregues e a quantidade de pedidos cancelados.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    SUM(total_final) AS faturamento_geral,
    SUM(total_final) FILTER (WHERE status = 'entregue') AS faturamento_entregue,
    COUNT(*) FILTER (WHERE status = 'cancelado') AS qtd_pedidos_cancelados
FROM pedidos;
```
</details>

---

#### Questão 17: Anti-Join para Registros Órfãos
Escreva uma consulta que liste todos os clientes que **nunca realizaram nenhum pedido** utilizando o padrão `LEFT JOIN ... WHERE IS NULL`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT c.id, c.nome, c.email
FROM clientes c
LEFT JOIN pedidos p ON c.id = p.cliente_id
WHERE p.id IS NULL;
```
</details>

---

#### Questão 18: Consulta Semi-Estruturada em JSONB
Dada a tabela `clientes` com a coluna `endereco` em JSONB, busque todos os clientes que residem na cidade de `'Curitiba'` e no estado (`uf`) de `'PR'`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
-- Opção 1: Operador ->> (como texto)
SELECT nome, endereco
FROM clientes
WHERE endereco->>'cidade' = 'Curitiba' AND endereco->>'uf' = 'PR';

-- Opção 2: Operador @> (contém objeto - ótimo com índice GIN!)
SELECT nome, endereco
FROM clientes
WHERE endereco @> '{"cidade": "Curitiba", "uf": "PR"}';
```
</details>

---

#### Questão 19: Concatenando Linhas em Lista com STRING_AGG
Liste para cada cliente o seu nome e uma coluna contendo todos os IDs dos seus pedidos separados por vírgula em ordem crescente.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    c.nome,
    STRING_AGG(p.id::TEXT, ', ' ORDER BY p.id ASC) AS lista_pedidos
FROM clientes c
JOIN pedidos p ON c.id = p.cliente_id
GROUP BY c.id, c.nome;
```
</details>

---

#### Questão 20: Auto-Relacionamento com SELF JOIN
Dada a tabela `categorias` com colunas `id`, `nome` e `categoria_pai_id`, escreva uma consulta que exiba o nome da subcategoria e o nome de sua respectiva categoria mãe.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    sub.nome AS subcategoria,
    COALESCE(mae.nome, '[Categoria Raiz]') AS categoria_pai
FROM categorias sub
LEFT JOIN categorias mae ON sub.categoria_pai_id = mae.id;
```
</details>

---

### 🔴 Nível 3: SQL Avançado, Performance, Window Functions e Neon (Questões 21 a 30)

#### Questão 21: View Materializada com Atualização Concorrente
Crie uma Materialized View `mv_relatorio_diario_vendas` e demonstre como configurá-la para permitir `REFRESH ... CONCURRENTLY`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
-- 1. Criação da View Materializada
CREATE MATERIALIZED VIEW mv_relatorio_diario_vendas AS
SELECT 
    data_pedido::DATE AS dia,
    COUNT(*) AS total_pedidos,
    SUM(total_final) AS faturamento
FROM pedidos
GROUP BY data_pedido::DATE;

-- 2. Índice único obrigatório para atualização concorrente
CREATE UNIQUE INDEX idx_mv_vendas_dia ON mv_relatorio_diario_vendas (dia);

-- 3. Atualização sem bloquear consultas de leitura
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_relatorio_diario_vendas;
```
</details>

---

#### Questão 22: Window Function para Ranking de Melhores Vendedores
Utilizando a função `DENSE_RANK()`, liste os vendedores exibindo seu nome, filial, faturamento e seu ranking de vendas **dentro de sua própria filial**.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    vendedor,
    filial,
    valor,
    DENSE_RANK() OVER (
        PARTITION BY filial 
        ORDER BY valor DESC
    ) AS posicao_na_filial
FROM vendas_filiais;
```
</details>

---

#### Questão 23: Cálculo de Variação Mês a Mês com LAG()
Calcule a variação percentual de faturamento do mês atual comparado ao mês anterior utilizando a função de deslocamento `LAG()`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
WITH faturamento_mensal AS (
    SELECT 
        DATE_TRUNC('month', data_pedido) AS mes,
        SUM(total_final) AS total_mes
    FROM pedidos
    GROUP BY DATE_TRUNC('month', data_pedido)
)
SELECT 
    mes,
    total_mes,
    LAG(total_mes, 1) OVER (ORDER BY mes) AS mes_anterior,
    ROUND(
        ((total_mes - LAG(total_mes, 1) OVER (ORDER BY mes)) / 
        LAG(total_mes, 1) OVER (ORDER BY mes)) * 100, 
        2
    ) AS pct_crescimento_mom
FROM faturamento_mensal;
```
</details>

---

#### Questão 24: CTE Recursiva para Gerar Séries Temporais
Utilize uma CTE Recursiva (`WITH RECURSIVE`) para gerar uma sequência contendo todos os primeiros dias de cada mês do ano de 2026.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
WITH RECURSIVE meses_2026 AS (
    -- Caso base: 1º de janeiro de 2026
    SELECT '2026-01-01'::DATE AS primeiro_dia
    
    UNION ALL
    
    -- Adiciona 1 mês até dezembro
    SELECT (primeiro_dia + INTERVAL '1 month')::DATE
    FROM meses_2026
    WHERE primeiro_dia < '2026-12-01'::DATE
)
SELECT primeiro_dia FROM meses_2026;
```
</details>

---

#### Questão 25: Trigger de Auditoria Automática
Crie uma tabela `logs_alteracao_salario` e uma Trigger em PL/pgSQL que registre o salário antigo e o novo salário sempre que houver um `UPDATE` na coluna `salario` de um funcionário.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
CREATE TABLE logs_alteracao_salario (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    funcionario_id INT NOT NULL,
    salario_anterior NUMERIC(10,2) NOT NULL,
    salario_novo NUMERIC(10,2) NOT NULL,
    data_alteracao TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION fn_auditar_salario()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF OLD.salario <> NEW.salario THEN
        INSERT INTO logs_alteracao_salario (funcionario_id, salario_anterior, salario_novo)
        VALUES (OLD.id, OLD.salario, NEW.salario);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER tg_auditoria_salario
    AFTER UPDATE OF salario ON funcionarios
    FOR EACH ROW
    EXECUTE FUNCTION fn_auditar_salario();
```
</details>

---

#### Questão 26: Evitando Concorrência com FOR UPDATE SKIP LOCKED
Escreva a consulta que um worker de backend deve executar para selecionar e travar exatamente 1 tarefa com status `'aguardando'` sem bloquear outros workers concorrentes.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
BEGIN;

SELECT id, payload
FROM fila_processamento
WHERE status = 'aguardando'
ORDER BY prioridade DESC, id ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- Processa no backend e finaliza:
UPDATE fila_processamento SET status = 'concluido' WHERE id = :id_obtido;

COMMIT;
```
</details>

---

#### Questão 27: Análise de Custo com EXPLAIN ANALYZE
Dado o plano de execução abaixo, responda:
```text
Seq Scan on logs (cost=0.00..4500.00 rows=150000 width=32) (actual time=0.021..142.100 rows=1 loops=1)
  Filter: (codigo_rastreio = 'BR123456789')
  Rows Removed by Filter: 149999
```
Qual é o gargalo de desempenho e qual comando SQL resolve definitivamente o problema?

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

* **Gargalo**: O PostgreSQL realizou um `Seq Scan` (varredura sequencial), lendo 150.000 linhas do disco para retornar apenas 1 única linha (`Rows Removed by Filter: 149999`), levando 142 ms.
* **Solução**: Criar um índice B-Tree na coluna `codigo_rastreio`:
```sql
CREATE INDEX CONCURRENTLY idx_logs_rastreio ON logs (codigo_rastreio);
```
*Com o índice, o tempo cairá de 142 ms para menos de 0.5 ms com um `Index Scan`.*
</details>

---

#### Questão 28: Criação de Branch no Neon via Linha de Comando
Como professor, qual comando da CLI `neonctl` você deve executar para criar um novo branch de banco de dados chamado `exercicio-turma-b` clonado a partir do branch `main`?

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```bash
neonctl branches create --name exercicio-turma-b --parent main
```
</details>

---

#### Questão 29: Point-in-Time Recovery no Neon
Um comando `DROP TABLE pedidos` foi acidentalmente executado às `2026-10-07 14:15:30 UTC`. Escreva o comando `neonctl` para criar um branch resgatando o banco exatamente 30 segundos antes do incidente.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```bash
neonctl branches create \
  --name resgate-acidente \
  --parent main \
  --time "2026-10-07T14:15:00Z"
```
</details>

---

#### Questão 30: Busca Semântica de Vetores com pgvector
Dada a tabela `artigos` com a coluna `vetor_conteudo VECTOR(1536)`, escreva a consulta SQL para retornar os 5 artigos semanticamente mais similares ao vetor de consulta `'[0.021, -0.043, ...]'`.

<details>
<summary>💡 Ver Gabarito e Explicação</summary>

```sql
SELECT 
    id,
    titulo,
    vetor_conteudo <=> '[0.021, -0.043, ...]' AS distancia_cosseno
FROM artigos
ORDER BY distancia_cosseno ASC
LIMIT 5;
```
*O operador `<=>` calcula a distância de cosseno. Ordenando de forma crescente (`ASC`), os 5 itens com menor distância são os mais similares.*
</details>

---
> **Navegação**: [⬅️ Aula Anterior: Projeto E-Commerce](./01-projeto-ecommerce.md) | [Módulo 06](./README.md) | [Módulo 07: Guias de Referência ➡️](../07-guias-de-referencia-rapida/README.md)
