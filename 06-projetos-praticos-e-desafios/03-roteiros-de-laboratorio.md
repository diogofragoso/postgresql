# 🚀 Módulo 06: Projetos Práticos e Desafios
## 🔬 Roteiros de Laboratório Prático para Sala de Aula

> **Navegação**: [⬅️ Banco de Questões](./02-banco-de-questoes-e-exercicios.md) | [Módulo 06](./README.md) | [Módulo 07: Guias de Referência ➡️](../07-guias-de-referencia-rapida/README.md)

---

### 🎯 Sobre Estes Roteiros
Estes 5 laboratórios práticos foram estruturados para **sessões práticas de 45 a 60 minutos** em sala de aula, laboratório de informática ou estudos individuais. Cada laboratório traz um cenário do mundo real, tarefas guiadas passo a passo e o gabarito completo em abas retráteis.

---

### 🧪 Laboratório 1: Modelagem e Integridade de Dados em uma Fintech
* **Duração Recomendada**: 50 minutos
* **Objetivo**: Aplicar tipos modernos (`UUID`, `NUMERIC`, `TIMESTAMPTZ`), chaves estrangeiras com ações referenciais e `CHECK constraints` matemáticas defensivas.

#### Cenário de Negócio:
Uma startup financeira precisa criar a estrutura inicial para controle de contas correntes e transações entre usuários. O sistema deve impedir saldos negativos não autorizados e não pode permitir transferências com valor zero ou negativo.

#### Tarefas a Executar:
1. Crie uma tabela `contas_correntes` com:
   - `id`: UUID gerado automaticamente.
   - `titular`: Texto não nulo.
   - `cpf`: 11 caracteres únicos.
   - `saldo`: Numérico com 2 casas decimais, não nulo, valor padrão `0.00` e restrição garantindo que nunca seja negativo (`saldo >= 0`).
2. Crie uma tabela `movimentacoes` com:
   - `id`: Inteiro com identidade SQL padrão moderno.
   - `conta_origem_id`: Chave estrangeira referenciando a conta de origem (`ON DELETE RESTRICT`).
   - `conta_destino_id`: Chave estrangeira referenciando a conta de destino (`ON DELETE RESTRICT`).
   - `valor`: Numérico com 2 casas decimais, restrição obrigando valor estritamente maior que zero (`valor > 0`).
   - `efetuada_em`: Timestamp com timezone padrão atual.
3. Insira duas contas com saldos iniciais de R$ 1.000,00 e R$ 500,00.
4. Tente intencionalmente inserir uma transferência com valor `-50.00` e observe o erro da Check Constraint.
5. Execute uma transferência válida de R$ 150,00 da conta 1 para a conta 2 dentro de uma transação (`BEGIN / COMMIT`).

<details>
<summary>💡 Ver Gabarito Completo do Laboratório 1</summary>

```sql
-- 1. Criação da tabela de contas
CREATE TABLE contas_correntes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    titular TEXT NOT NULL,
    cpf VARCHAR(11) NOT NULL UNIQUE,
    saldo NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    criada_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_cpf_formato CHECK (length(cpf) = 11),
    CONSTRAINT chk_saldo_positivo CHECK (saldo >= 0.00)
);

-- 2. Criação da tabela de movimentações
CREATE TABLE movimentacoes (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    conta_origem_id UUID NOT NULL REFERENCES contas_correntes(id) ON DELETE RESTRICT,
    conta_destino_id UUID NOT NULL REFERENCES contas_correntes(id) ON DELETE RESTRICT,
    valor NUMERIC(12,2) NOT NULL,
    efetuada_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_valor_positivo CHECK (valor > 0.00),
    CONSTRAINT chk_contas_distintas CHECK (conta_origem_id <> conta_destino_id)
);

-- 3. Inserção de contas iniciais
INSERT INTO contas_correntes (titular, cpf, saldo) VALUES
    ('Mariana Lima', '12345678901', 1000.00),
    ('Felipe Santos', '98765432100', 500.00);

-- 4. Teste de violação de regra (Deve falhar com erro de CHECK constraint!):
-- INSERT INTO movimentacoes (conta_origem_id, conta_destino_id, valor)
-- SELECT c1.id, c2.id, -50.00
-- FROM contas_correntes c1, contas_correntes c2
-- WHERE c1.titular = 'Mariana Lima' AND c2.titular = 'Felipe Santos';

-- 5. Transferência atômica segura
DO $$
DECLARE
    v_origem UUID;
    v_destino UUID;
BEGIN
    SELECT id INTO v_origem FROM contas_correntes WHERE titular = 'Mariana Lima';
    SELECT id INTO v_destino FROM contas_correntes WHERE titular = 'Felipe Santos';

    -- Debita da conta de origem
    UPDATE contas_correntes SET saldo = saldo - 150.00 WHERE id = v_origem;
    
    -- Credita na conta de destino
    UPDATE contas_correntes SET saldo = saldo + 150.00 WHERE id = v_destino;

    -- Registra o log da movimentação
    INSERT INTO movimentacoes (conta_origem_id, conta_destino_id, valor)
    VALUES (v_origem, v_destino, 150.00);
END $$;

-- Verificação final
SELECT titular, saldo FROM contas_correntes;
```
</details>

---

### 🧪 Laboratório 2: DML e Padrão UPSERT em Catálogo de Streaming
* **Duração Recomendada**: 45 minutos
* **Objetivo**: Dominar inserções com `RETURNING`, atualizações com `ON CONFLICT DO UPDATE` e filtros avançados com busca case-insensitive (`ILIKE`).

#### Cenário de Negócio:
Uma plataforma de streaming precisa cadastrar filmes e manter um ranking atualizado de visualizações (*views*) por usuário sem gerar linhas duplicadas.

#### Tarefas a Executar:
1. Crie a tabela `catalogo_filmes` (`id`, `titulo`, `genero`, `ano_lancamento`).
2. Crie a tabela `historico_visualizacoes` com chave única composta por `(usuario_id, filme_id)`, além de `total_views` (int) e `ultima_visualizacao` (timestamptz).
3. Insira 4 filmes e retorne os IDs gerados usando `RETURNING`.
4. Escreva uma instrução de **UPSERT**: se o usuário 10 assistir ao filme 1 pela primeira vez, insira com `total_views = 1`; se já tiver assistido, incremente `total_views = total_views + 1` e atualize a data.
5. Execute a instrução 3 vezes consecutivas e verifique o contador incrementando.
6. Faça uma busca por filmes de ficção científica lançados após o ano 2020 cujo título contenha a palavra `'star'` (insensível a maiúsculas).

<details>
<summary>💡 Ver Gabarito Completo do Laboratório 2</summary>

```sql
-- 1. Catálogo
CREATE TABLE catalogo_filmes (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo TEXT NOT NULL,
    genero TEXT NOT NULL,
    ano_lancamento INT NOT NULL
);

-- 2. Histórico com chave única
CREATE TABLE historico_visualizacoes (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    usuario_id INT NOT NULL,
    filme_id INT NOT NULL REFERENCES catalogo_filmes(id) ON DELETE CASCADE,
    total_views INT NOT NULL DEFAULT 1,
    ultima_visualizacao TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_usuario_filme UNIQUE (usuario_id, filme_id)
);

-- 3. Inserção retornando IDs
INSERT INTO catalogo_filmes (titulo, genero, ano_lancamento) VALUES
    ('Interstellar', 'Ficção Científica', 2014),
    ('Star Wars: Nova Esperança', 'Ficção Científica', 1977),
    ('Duna: Parte 2', 'Ficção Científica', 2024),
    ('Star Trek: Além do Infinito', 'Ficção Científica', 2022)
RETURNING id, titulo;

-- 4 e 5. UPSERT executado 3 vezes:
INSERT INTO historico_visualizacoes (usuario_id, filme_id, total_views, ultima_visualizacao)
VALUES (10, 1, 1, CURRENT_TIMESTAMP)
ON CONFLICT (usuario_id, filme_id)
DO UPDATE SET
    total_views = historico_visualizacoes.total_views + 1,
    ultima_visualizacao = CURRENT_TIMESTAMP
RETURNING usuario_id, filme_id, total_views, ultima_visualizacao;

-- 6. Busca com ILIKE e filtro de ano
SELECT * FROM catalogo_filmes
WHERE genero = 'Ficção Científica'
  AND ano_lancamento >= 2020
  AND titulo ILIKE '%star%';
```
</details>

---

### 🧪 Laboratório 3: Inteligência de Negócios (BI) com Agregações e Window Functions
* **Duração Recomendada**: 60 minutos
* **Objetivo**: Extrair métricas corporativas avançadas utilizando a cláusula `FILTER`, rankings densos (`DENSE_RANK`) e comparação temporal mês a mês com `LAG()`.

#### Tarefas a Executar:
1. Utilize a base de dados de vendas abaixo:
```sql
CREATE TABLE vendas_mensais_equipe (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mes_ano DATE NOT NULL,
    regiao TEXT NOT NULL,
    vendedor TEXT NOT NULL,
    valor_faturado NUMERIC(10,2) NOT NULL
);

INSERT INTO vendas_mensais_equipe (mes_ano, regiao, vendedor, valor_faturado) VALUES
    ('2026-01-01', 'Sul', 'Lucas', 15000.00),
    ('2026-01-01', 'Sul', 'Beatriz', 22000.00),
    ('2026-01-01', 'Sudeste', 'Rodrigo', 31000.00),
    ('2026-01-01', 'Sudeste', 'Camila', 28000.00),
    ('2026-02-01', 'Sul', 'Lucas', 18000.00),
    ('2026-02-01', 'Sul', 'Beatriz', 24500.00),
    ('2026-02-01', 'Sudeste', 'Rodrigo', 29000.00),
    ('2026-02-01', 'Sudeste', 'Camila', 35000.00);
```
2. **Relatório 1 (Cláusula FILTER)**: Agrupe por mês e exiba o faturamento total da empresa, o faturamento exclusivo da região Sul e o faturamento exclusivo da região Sudeste em colunas separadas.
3. **Relatório 2 (Window Function Ranking)**: Liste cada vendedor, sua região e seu ranking de faturamento dentro da própria região para o mês de fevereiro de 2026 (`DENSE_RANK`).
4. **Relatório 3 (Crescimento MoM com LAG)**: Agrupe o faturamento total da empresa mês a mês e calcule a variação percentual de crescimento em relação ao mês anterior.

<details>
<summary>💡 Ver Gabarito Completo do Laboratório 3</summary>

```sql
-- Relatório 1: Agrupamento com FILTER
SELECT 
    mes_ano,
    SUM(valor_faturado) AS faturamento_global,
    SUM(valor_faturado) FILTER (WHERE regiao = 'Sul') AS faturamento_sul,
    SUM(valor_faturado) FILTER (WHERE regiao = 'Sudeste') AS faturamento_sudeste
FROM vendas_mensais_equipe
GROUP BY mes_ano
ORDER BY mes_ano;

-- Relatório 2: Ranking por Região com DENSE_RANK
SELECT 
    vendedor,
    regiao,
    valor_faturado,
    DENSE_RANK() OVER (
        PARTITION BY regiao 
        ORDER BY valor_faturado DESC
    ) AS posicao_na_regiao
FROM vendas_mensais_equipe
WHERE mes_ano = '2026-02-01';

-- Relatório 3: Variação Mês a Mês (Month-over-Month) com LAG
WITH resumo_meses AS (
    SELECT 
        mes_ano,
        SUM(valor_faturado) AS total_mes
    FROM vendas_mensais_equipe
    GROUP BY mes_ano
)
SELECT 
    mes_ano,
    total_mes,
    LAG(total_mes, 1) OVER (ORDER BY mes_ano) AS mes_anterior,
    ROUND(
        ((total_mes - LAG(total_mes, 1) OVER (ORDER BY mes_ano)) / 
        LAG(total_mes, 1) OVER (ORDER BY mes_ano)) * 100, 
        2
    ) AS pct_crescimento_mom
FROM resumo_meses;
```
</details>

---

### 🧪 Laboratório 4: Automação e Auditoria com Triggers em PL/pgSQL
* **Duração Recomendada**: 50 minutos
* **Objetivo**: Criar gatilhos para validar regras de negócio antes da gravação (`BEFORE`) e gravar logs de auditoria históricos automaticamente (`AFTER`).

#### Tarefas a Executar:
1. Crie uma tabela `produtos_loja` com `id`, `nome`, `preco_venda` e `atualizado_em`.
2. Crie uma tabela `auditoria_precos_log` com `produto_id`, `preco_antigo`, `preco_novo`, `alterado_por` e `data_alteracao`.
3. Escreva um Trigger `BEFORE UPDATE` que atualize automaticamente a coluna `atualizado_em` para `CURRENT_TIMESTAMP`.
4. Escreva um Trigger `AFTER UPDATE` que verifique se o `preco_venda` foi alterado. Se sim, grave uma linha de auditoria com os valores de `OLD.preco_venda` e `NEW.preco_venda`.
5. Insira um produto, altere o preço 2 vezes e consulte a tabela de auditoria para confirmar os logs registrados.

<details>
<summary>💡 Ver Gabarito Completo do Laboratório 4</summary>

```sql
-- 1 e 2. Tabelas
CREATE TABLE produtos_loja (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome TEXT NOT NULL,
    preco_venda NUMERIC(10,2) NOT NULL,
    atualizado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE auditoria_precos_log (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    produto_id INT NOT NULL,
    preco_antigo NUMERIC(10,2) NOT NULL,
    preco_novo NUMERIC(10,2) NOT NULL,
    alterado_por TEXT DEFAULT CURRENT_USER,
    data_alteracao TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 3. Função e Trigger para atualizado_em (BEFORE)
CREATE OR REPLACE FUNCTION fn_atualizar_timestamp()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.atualizado_em = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;

CREATE TRIGGER tg_produtos_timestamp
    BEFORE UPDATE ON produtos_loja
    FOR EACH ROW
    EXECUTE FUNCTION fn_atualizar_timestamp();

-- 4. Função e Trigger para auditoria de preço (AFTER)
CREATE OR REPLACE FUNCTION fn_auditar_preco()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF OLD.preco_venda <> NEW.preco_venda THEN
        INSERT INTO auditoria_precos_log (produto_id, preco_antigo, preco_novo)
        VALUES (OLD.id, OLD.preco_venda, NEW.preco_venda);
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER tg_produtos_auditoria
    AFTER UPDATE ON produtos_loja
    FOR EACH ROW
    EXECUTE FUNCTION fn_auditar_preco();

-- 5. Testes práticos
INSERT INTO produtos_loja (nome, preco_venda) VALUES ('Teclado Mecânico', 250.00);

-- Primeira alteração de preço
UPDATE produtos_loja SET preco_venda = 279.90 WHERE id = 1;

-- Segunda alteração de preço
UPDATE produtos_loja SET preco_venda = 249.00 WHERE id = 1;

-- Verificando a auditoria automática gravada pelo Postgres
SELECT * FROM auditoria_precos_log;
```
</details>

---

### 🧪 Laboratório 5: Nuvem Serverless, Database Branching e Busca Semântica de IA no Neon
* **Duração Recomendada**: 50 minutos
* **Objetivo**: Praticar branches efêmeros para testes sem risco de produção e utilizar a extensão nativa `pgvector` para buscas semânticas vetoriais.

#### Tarefas a Executar:
1. No seu banco do Neon, crie um branch chamado `lab-ia-vetores`.
2. Conecte-se ao branch e ative a extensão `vector` com `CREATE EXTENSION IF NOT EXISTS vector;`.
3. Crie uma tabela `catalogo_livros_ia` com:
   - `id`: Inteiro identity.
   - `titulo`: Texto.
   - `categoria`: Texto.
   - `embedding`: Coluna `VECTOR(3)` (vetor tridimensional para fins didáticos).
4. Insira 4 livros com vetores simulando seus tópicos conceituais.
5. Escreva uma busca semântica para encontrar o livro mais relevante em relação ao vetor de busca `'[0.91, 0.12, 0.05]'` utilizando o operador de distância de cosseno `<=>`.
6. Após validar os testes, exclua o branch de teste no painel do Neon ou via CLI.

<details>
<summary>💡 Ver Gabarito Completo do Laboratório 5</summary>

```sql
-- 1 e 2. Ativar a extensão de vetores
CREATE EXTENSION IF NOT EXISTS vector;

-- 3. Criação da tabela com campo vetorial
CREATE TABLE catalogo_livros_ia (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo TEXT NOT NULL,
    categoria TEXT NOT NULL,
    embedding VECTOR(3) NOT NULL
);

-- 4. Inserção de dados com embeddings
-- Vetores próximos representam conceitos similares no espaço semântico
INSERT INTO catalogo_livros_ia (titulo, categoria, embedding) VALUES
    ('PostgreSQL Essencial e Alta Performance', 'Banco de Dados', '[0.90, 0.10, 0.05]'),
    ('Guia Definitivo de Docker e Kubernetes', 'Infraestrutura', '[0.82, 0.25, 0.10]'),
    ('Culinária Italiana Tradicional', 'Gastronomia', '[0.05, 0.95, 0.15]'),
    ('Receitas Rápidas de Massas e Molhos', 'Gastronomia', '[0.08, 0.92, 0.20]');

-- 5. Busca semântica por proximidade de Cosseno (<=>)
-- Buscando conteúdo mais próximo do vetor [0.91, 0.12, 0.05] (Tema de Bancos/Dados):
SELECT 
    titulo,
    categoria,
    embedding <=> '[0.91, 0.12, 0.05]' AS distancia_cosseno
FROM catalogo_livros_ia
ORDER BY distancia_cosseno ASC
LIMIT 2;
```

*Os dois livros técnicos aparecerão no topo com distância próxima de zero, enquanto os livros de culinária ficarão no final da lista!*
</details>

---
> **Navegação**: [⬅️ Banco de Questões](./02-banco-de-questoes-e-exercicios.md) | [Módulo 06](./README.md) | [Módulo 07: Guias de Referência ➡️](../07-guias-de-referencia-rapida/README.md)
