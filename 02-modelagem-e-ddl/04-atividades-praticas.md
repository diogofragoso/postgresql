# 🛠️ Atividades Práticas: Módulo 02 - Modelagem e DDL

<div align="center">

![Nível](https://img.shields.io/badge/Nível-Iniciante_ao_Intermediário-green?style=for-the-badge)
![Tipo](https://img.shields.io/badge/Tipo-Laboratório_Prático-orange?style=for-the-badge)
![Ambiente](https://img.shields.io/badge/Ambiente-Ubuntu_Docker_ou_Neon_Beekeeper-blue?style=for-the-badge)

</div>

Este caderno reúne as atividades práticas do **Módulo 02**. Todas as atividades estão organizadas em subitens independentes com enunciados desafiadores, instruções passo a passo e gabaritos comentados com a tag retrátil `<details>`.

---

## 📑 Índice de Subitens Práticos

1. [Subitem 2.1: Modelagem Relacional e Normalização de Sistema Escolar](#-subitem-21-modelagem-relacional-e-normalizacao-de-sistema-escolar)
2. [Subitem 2.2: Laboratório de Tipos Modernos com UUID e JSONB para Sensores IoT](#-subitem-22-laboratorio-de-tipos-modernos-com-uuid-e-jsonb-para-sensores-iot)
3. [Subitem 2.3: Criação de Schema e Defesa de Integridade com Constraints Rígidas](#-subitem-23-criacao-de-schema-e-defesa-de-integridade-com-constraints-rigidas)

---

## 📌 Subitem 2.1: Modelagem Relacional e Normalização de Sistema Escolar

### 🎯 Objetivo
Aplicar as regras da Primeira, Segunda e Terceira Formas Normais (1FN, 2FN e 3FN) para transformar uma planilha de notas desnormalizada em uma arquitetura relacional de banco de dados robusta, com relacionamentos 1:N e N:N.

### 💻 Ambiente
- **Trilha A**: Terminal `psql` ou Beekeeper Studio Portable conectado ao Docker.
- **Trilha B**: Beekeeper Studio Portable ou Neon SQL Editor.

### 📋 Enunciado e Desafio
Considere o seguinte cenário: Uma escola mantém uma planilha com os campos:
`[aluno_cpf, aluno_nome, curso_nome, disciplina_nome, professor_nome, nota, data_avaliacao]`.

1. Identifique as violações de 1FN, 2FN e 3FN presentes nesta estrutura plana.
2. Crie um modelo relacional normalizado contendo no mínimo 4 tabelas:
   - `alunos` (dados cadastrais do estudante).
   - `cursos` (cursos oferecidos).
   - `disciplinas` (matérias vinculadas a cursos).
   - `matriculas_notas` (relação N:N de notas de alunos em disciplinas).
3. Escreva o script DDL no PostgreSQL criando todas as tabelas com suas respectivas chaves primárias e estrangeiras.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 2.1</summary>

```sql
-- 1. Criação das tabelas normalizadas em 3FN

-- Entidade Alunos
CREATE TABLE alunos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cpf CHAR(11) NOT NULL UNIQUE,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    criado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Entidade Cursos
CREATE TABLE cursos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome VARCHAR(100) NOT NULL UNIQUE,
    carga_horaria INT NOT NULL CHECK (carga_horaria > 0)
);

-- Entidade Disciplinas (Pertence a um Curso)
CREATE TABLE disciplinas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    curso_id INT NOT NULL REFERENCES cursos(id) ON DELETE CASCADE,
    nome VARCHAR(100) NOT NULL,
    codigo CHAR(6) NOT NULL UNIQUE
);

-- Relacionamento N:N de Avaliações
CREATE TABLE matriculas_notas (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aluno_id INT NOT NULL REFERENCES alunos(id) ON DELETE CASCADE,
    disciplina_id INT NOT NULL REFERENCES disciplinas(id) ON DELETE CASCADE,
    nota NUMERIC(4,2) NOT NULL CHECK (nota >= 0.00 AND nota <= 10.00),
    data_avaliacao DATE NOT NULL DEFAULT CURRENT_DATE,
    CONSTRAINT uq_aluno_disciplina_data UNIQUE (aluno_id, disciplina_id, data_avaliacao)
);
```

**Explicação Pedagógica:**
- **1FN**: Cada célula agora é atômica, eliminando listas ou múltiplos valores por coluna.
- **2FN**: Todos os atributos não-chave dependem inteiramente da chave primária das tabelas.
- **3FN**: Não existem dependências transitivas (o nome do curso não fica na tabela de disciplinas nem de alunos).
</details>

---

## 📌 Subitem 2.2: Laboratório de Tipos Modernos com UUID e JSONB para Sensores IoT

### 🎯 Objetivo
Praticar o uso de **UUIDv4** nativo para geração de chaves primárias seguras e o tipo binário semiestruturado **JSONB** para dados variáveis e telemétricos de dispositivos inteligentes.

### 💻 Ambiente
- Beekeeper Studio Portable ou Neon Cloud / Docker Ubuntu.

### 📋 Enunciado e Desafio
1. Crie uma tabela `leituras_sensores` contendo:
   - `id`: Chave primária do tipo `UUID` gerada automaticamente via `gen_random_uuid()`.
   - `dispositivo_mac`: Endereço MAC do dispositivo (`macaddr` ou `VARCHAR(17)`).
   - `dados_telemetria`: Coluna do tipo `JSONB` com leituras flexíveis (temperatura, umidade, voltagem, alertas).
   - `registrado_em`: Timestamp com timezone (`TIMESTAMPTZ`).
2. Insira 3 registros simulando sensores de armazéns refrigerados com formatos de payload ligeiramente diferentes.
3. Escreva consultas SQL para:
   - Extrair a temperatura como número (`NUMERIC`) usando o operador `->>`.
   - Filtrar leituras onde o armazém seja `'armazem_01'` utilizando o operador de contenção JSONB `@>`.
   - Filtrar registros que possuem a chave booleana `"alerta_ativo": true`.

<details>
<summary>👁️ Clique aqui para ver o gabarito e explicação do Subitem 2.2</summary>

```sql
-- 1. Criação da tabela com UUID e JSONB
CREATE TABLE leituras_sensores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dispositivo_mac VARCHAR(17) NOT NULL,
    dados_telemetria JSONB NOT NULL,
    registrado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Inserção de dados de telemetria variados
INSERT INTO leituras_sensores (dispositivo_mac, dados_telemetria) VALUES
(
    '00:1B:44:11:3A:B7',
    '{"armazem": "armazem_01", "temperatura_c": -18.4, "umidade_pct": 65, "alerta_ativo": false}'::jsonb
),
(
    '00:1B:44:11:3A:B7',
    '{"armazem": "armazem_01", "temperatura_c": -5.1, "umidade_pct": 82, "alerta_ativo": true, "motivo": "Porta aberta"}'::jsonb
),
(
    '00:14:22:01:23:45',
    '{"armazem": "armazem_02", "temperatura_c": 4.2, "bateria_pct": 98, "alerta_ativo": false}'::jsonb
);

-- 3. Consulta A: Extrair temperatura convertida para número
SELECT 
    id,
    dispositivo_mac,
    (dados_telemetria ->> 'temperatura_c')::NUMERIC AS temp_celsius,
    registrado_em
FROM leituras_sensores
ORDER BY temp_celsius ASC;

-- 3. Consulta B: Filtrar com operador de contenção (@>)
SELECT 
    id,
    dados_telemetria ->> 'temperatura_c' AS temperatura,
    dados_telemetria ->> 'motivo' AS motivo_alerta
FROM leituras_sensores
WHERE dados_telemetria @> '{"armazem": "armazem_01"}';

-- 3. Consulta C: Buscar sensores em estado de alerta crítico
SELECT 
    id,
    dispositivo_mac,
    dados_telemetria ->> 'motivo' AS motivo_problema
FROM leituras_sensores
WHERE dados_telemetria @> '{"alerta_ativo": true}';
```

**Explicação Pedagógica:**
- O operador `->>` extrai o valor interno do JSON como texto (`TEXT`), permitindo o cast para números com `::NUMERIC`.
- O operador `@>` (*contains*) verifica se o JSON da esquerda contém a estrutura do JSON da direita. Essa operação é altamente otimizável no PostgreSQL através de índices GIN (`USING gin (dados_telemetria)`).
</details>

---

## 📌 Subitem 2.3: Criação de Schema e Defesa de Integridade com Constraints Rígidas

### 🎯 Objetivo
Aprender a delegar regras de negócio ao banco de dados através de constraints declarativas (`CHECK`, `UNIQUE`, `NOT NULL`, `FOREIGN KEY` e tipos enumerados `CREATE TYPE AS ENUM`), garantindo que dados inválidos nunca sejam persistidos.

### 💻 Ambiente
- Beekeeper Studio Portable ou psql.

### 📋 Enunciado e Desafio
Você foi contratado para criar a estrutura da tabela de **pedidos** de uma distribuidora:
1. Crie um tipo enumerado `status_pedido_enum` com os valores: `'pendente'`, `'pago'`, `'enviado'`, `'cancelado'`.
2. Crie a tabela `pedidos_venda` contendo:
   - `id`: Identificador sequencial com `GENERATED ALWAYS AS IDENTITY`.
   - `codigo_rastreio`: Código alfanumérico único no formato exato de 13 caracteres (ex: `BR123456789BR`), podendo ser nulo apenas se o pedido ainda não foi enviado.
   - `valor_subtotal`: Numérico maior que zero.
   - `valor_frete`: Numérico maior ou igual a zero.
   - `valor_desconto`: Numérico maior ou igual a zero, com a restrição de que o desconto não pode ser maior do que o `valor_subtotal`.
   - `status`: Do tipo `status_pedido_enum`, com padrão `'pendente'`.
3. Tente inserir intencionalmente um pedido com valor negativo ou com desconto superior ao subtotal e confirme a rejeição pelo motor PostgreSQL.

<details>
<summary>👁️ Clique aqui para ver o script SQL e a validação do Subitem 2.3</summary>

```sql
-- 1. Criação do tipo enumerado
DROP TYPE IF EXISTS status_pedido_enum CASCADE;
CREATE TYPE status_pedido_enum AS ENUM ('pendente', 'pago', 'enviado', 'cancelado');

-- 2. Criação da tabela com constraints avançadas
DROP TABLE IF EXISTS pedidos_venda;
CREATE TABLE pedidos_venda (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    codigo_rastreio VARCHAR(13) UNIQUE,
    valor_subtotal NUMERIC(10,2) NOT NULL CHECK (valor_subtotal > 0),
    valor_frete NUMERIC(10,2) NOT NULL DEFAULT 0.00 CHECK (valor_frete >= 0),
    valor_desconto NUMERIC(10,2) NOT NULL DEFAULT 0.00 CHECK (valor_desconto >= 0),
    status status_pedido_enum NOT NULL DEFAULT 'pendente',
    criado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    
    -- Restrição a nível de tabela: Desconto não pode exceder o subtotal
    CONSTRAINT chk_desconto_valido CHECK (valor_desconto <= valor_subtotal)
);

-- 3. Inserção válida de teste
INSERT INTO pedidos_venda (valor_subtotal, valor_frete, valor_desconto, status)
VALUES (200.00, 25.00, 30.00, 'pago');

-- 4. Teste de Violação 1: Valor Subtotal Negativo (Deve FALHAR)
-- INSERT INTO pedidos_venda (valor_subtotal) VALUES (-50.00);
-- Erro esperado: new row for relation "pedidos_venda" violates check constraint "pedidos_venda_valor_subtotal_check"

-- 5. Teste de Violação 2: Desconto maior que o subtotal (Deve FALHAR)
-- INSERT INTO pedidos_venda (valor_subtotal, valor_desconto) VALUES (100.00, 150.00);
-- Erro esperado: violates check constraint "chk_desconto_valido"
```

**Explicação Pedagógica:**
- A integridade de dados aplicada na camada de banco previne inconsistências mesmo se múltiplos sistemas (APIs em Node, Python, microsserviços) acessarem o banco sem validação completa na aplicação.
</details>

---

## 🧭 Navegação
* [⬅️ Voltar para o Módulo 02: Modelagem e DDL](./README.md)
* [Ir para o Módulo 03: Manipulação DML e Consultas ➡️](../03-manipulacao-dml-e-consultas/README.md)
