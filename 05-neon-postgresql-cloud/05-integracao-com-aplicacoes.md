# ⚡ Módulo 05: Neon Serverless PostgreSQL
## 📑 Aula 05: Integração com Aplicações Modernas (Node.js, Prisma, Drizzle, Python e IA)

> **Navegação**: [⬅️ Aula Anterior: Escala e PITR](./04-escala-e-alta-disponibilidade.md) | [Módulo 05](./README.md) | [Módulo 06: Projetos Práticos ➡️](../06-projetos-praticos-e-desafios/README.md)

---

### 🎯 Objetivos de Aprendizagem
Ao final desta aula, você será capaz de:
- Conectar aplicações serverless (Vercel, Cloudflare, AWS Lambda) usando o driver oficial **`@neondatabase/serverless`** via HTTP/WebSockets.
- Configurar o **Prisma ORM** com separação de URL de runtime (com pooling) e URL direta de migrações (`directUrl`).
- Integrar com o **Drizzle ORM** para alta performance e tipagem estrita em TypeScript.
- Conectar aplicações **Python** usando `psycopg` e `SQLAlchemy`.
- Ativar e utilizar a extensão **`pgvector`** no Neon para armazenar embeddings vetoriais e criar buscas semânticas de Inteligência Artificial.

---

### 1. O Driver Serverless: `@neondatabase/serverless`

Bancos relacionais tradicionais dependem de conexões TCP com estado mantido permanentemente. Em plataformas Edge (Cloudflare Workers, Vercel Edge), sockets TCP diretos muitas vezes são bloqueados ou lentos para negociar o handshake SSL.

O Neon criou um driver open source que permite enviar queries SQL empacotadas via **HTTP/Fetch** ou **WebSockets seguros**:

```mermaid
flowchart LR
    Edge[Cloudflare Worker / Vercel Edge] -->|Sub-millisecond HTTP/WS Query| NeonProxy[Neon Serverless WebSocket Proxy]
    NeonProxy -->|Protocolo Postgres Nativo| PG[(Compute Engine)]
```

#### Exemplo em TypeScript / Node.js:

```bash
npm install @neondatabase/serverless
```

```typescript
import { neon } from '@neondatabase/serverless';

// Inicializa o cliente apontando para a variável de ambiente
const sql = neon(process.env.DATABASE_URL!);

async function buscarUsuariosAtivos() {
  // Executa queries como uma Tagged Template Literal:
  const usuarios = await sql`
    SELECT id, nome, email 
    FROM usuarios 
    WHERE ativo = true 
    ORDER BY id DESC 
    LIMIT 10
  `;
  
  console.log(usuarios);
  return usuarios;
}
```

> [!TIP]
> O driver `@neondatabase/serverless` previne ataques de **SQL Injection** automaticamente ao parametrizar todas as variáveis interpoladas nos template literals!

---

### 2. Configurando Prisma ORM com o Neon

O Prisma é um dos ORMs mais populares do ecossistema TypeScript. Para funcionar com perfeição no Neon, configuramos duas URLs no arquivo `prisma/schema.prisma`:

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")        // Endpoint com -pooler (Para a aplicação rodando)
  directUrl = env("DIRECT_URL")          // Endpoint direto sem -pooler (Para o prisma migrate)
}

generator client {
  provider = "prisma-client-js"
}

model Usuario {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  nome      String
  criadoEm  DateTime @default(now())
}
```

#### Arquivo `.env`:
```env
# Runtime (Pooler com PgBouncer integrado):
DATABASE_URL="postgresql://alex:senha@ep-divine-pond-123456-pooler.us-east-2.aws.neon.tech/neondb?sslmode=require"

# Migrações DDL (Conexão direta ao backend):
DIRECT_URL="postgresql://alex:senha@ep-divine-pond-123456.us-east-2.aws.neon.tech/neondb?sslmode=require"
```

---

### 3. Integração com Drizzle ORM

O Drizzle ORM oferece uma camada leve e tipada com suporte nativo de primeira classe ao Neon via HTTP:

```bash
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit
```

```typescript
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';
import { pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core';

// Definindo o schema
export const produtos = pgTable('produtos', {
  id: serial('id').primaryKey(),
  nome: text('nome').notNull(),
  criadoEm: timestamp('criado_em').defaultNow(),
});

// Conectando
const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql);

// Consultando com tipagem completa:
const resultado = await db.select().from(produtos);
```

---

### 4. Integração com Python (Psycopg e SQLAlchemy)

No ecossistema Python, utilizamos o driver moderno `psycopg` (versão 3):

```bash
pip install "psycopg[binary]" sqlalchemy
```

```python
import psycopg

DATABASE_URL = "postgresql://alex:senha@ep-divine-pond-123456.us-east-2.aws.neon.tech/neondb?sslmode=require"

# Conexão direta com context manager seguro
with psycopg.connect(DATABASE_URL) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT id, nome, email FROM usuarios WHERE ativo = %s;", (True,))
        linhas = cur.fetchall()
        for linha in linhas:
            print(f"ID: {linha[0]} | Nome: {linha[1]} | E-mail: {linha[2]}")
```

---

### 5. Busca Semântica e IA: `pgvector` no Neon

O PostgreSQL e o Neon suportam a extensão **`pgvector`**, permitindo armazenar embeddings vetoriais (gerados por modelos como OpenAI text-embedding-3, Gemini ou Llama) e fazer busca por similaridade semântica diretamente em SQL!

```mermaid
flowchart LR
    Texto["Texto do Usuário: 'Tênis de corrida confortável'"] --> LLM[Modelo de IA / Embeddings]
    LLM --> Vetor["Vetor: [0.015, -0.042, 0.891, ...]"]
    Vetor -->|Busca por Distância Cosseno <=> | NeonPG[(PostgreSQL + pgvector)]
    NeonPG --> TopResultados["Produtos semanticamente mais relevantes"]
```

#### Ativando e Testando a Busca Vetorial:

```sql
-- 1. Ativar a extensão
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Criar tabela de artigos com vetor de 3 dimensões (simplificado para exemplo)
CREATE TABLE documentos_ia (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    conteudo TEXT NOT NULL,
    embedding VECTOR(3) -- Na prática, use 1536 (OpenAI) ou 768 (Gemini)
);

-- 3. Inserir documentos com seus vetores
INSERT INTO documentos_ia (conteudo, embedding) VALUES
    ('Guia completo de PostgreSQL e SQL', '[0.9, 0.1, 0.1]'),
    ('Como cozinhar uma lasanha tradicional', '[0.1, 0.8, 0.2]'),
    ('Apostila de administração de bancos de dados', '[0.85, 0.15, 0.12]');

-- 4. Busca Semântica por proximidade de Cosseno (Operador <=>):
-- Buscando o documento mais próximo do conceito [0.88, 0.12, 0.09]:
SELECT 
    conteudo,
    embedding <=> '[0.88, 0.12, 0.09]' AS distancia_cosseno
FROM documentos_ia
ORDER BY distancia_cosseno ASC
LIMIT 2;
```

> [!NOTE]
> Quanto menor a distância do cosseno (`<=>`), mais semanticamente próximo o documento está da busca! Essa técnica alimenta sistemas modernos de **RAG (Retrieval-Augmented Generation)**.

---

### 6. Atividade Prática 08: Busca Semântica e Indexação HNSW com pgvector

Nesta atividade, você irá aprofundar o uso de Inteligência Artificial no PostgreSQL, calculando a porcentagem de similaridade e acelerando a busca com o algoritmo moderno **HNSW (Hierarchical Navigable Small World)**.

#### 🎯 Desafio Prático:
1. Com base na tabela `documentos_ia` criada acima, crie um índice vetorial do tipo **HNSW** na coluna `embedding` usando a métrica de distância do cosseno (`vector_cosine_ops`).
2. Escreva uma consulta que receba um vetor de pergunta `[0.85, 0.12, 0.10]` e retorne:
   - O conteúdo do documento.
   - A distância do cosseno bruta.
   - O **score de similaridade percentual** calculado como: `ROUND(((1 - (embedding <=> '[0.85, 0.12, 0.10]')) * 100)::numeric, 2) AS similaridade_pct`.
3. Filtre para trazer apenas documentos com similaridade superior a 80% e ordene pelo mais relevante.

<details>
<summary>👁️ Clique aqui para ver o script SQL da solução</summary>

```sql
-- 1. Criação do índice de alta performance HNSW
CREATE INDEX IF NOT EXISTS idx_documentos_ia_hnsw 
ON documentos_ia 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 2. Consulta de Busca Semântica com score percentual
SELECT 
    conteudo,
    ROUND((embedding <=> '[0.85, 0.12, 0.10]')::numeric, 4) AS distancia_bruta,
    ROUND(((1 - (embedding <=> '[0.85, 0.12, 0.10]')) * 100)::numeric, 2) AS similaridade_pct
FROM documentos_ia
WHERE (1 - (embedding <=> '[0.85, 0.12, 0.10]')) >= 0.80
ORDER BY embedding <=> '[0.85, 0.12, 0.10]' ASC
LIMIT 5;
```

**Explicação Pedagógica:**
- O índice **HNSW** constrói um grafo de múltiplas camadas navegáveis, permitindo buscas vetoriais aproximadas (ANN - Approximate Nearest Neighbor) em milissegundos mesmo com milhões de registros.
- A fórmula `1 - distancia_cosseno` converte a distância (onde 0 é idêntico) em similaridade de cosseno (onde 1 ou 100% é idêntico).
</details>

---

### 📝 Checklist de Conclusão da Aula

- [ ] Compreendi a vantagem do driver `@neondatabase/serverless` em arquiteturas Edge.
- [ ] Sei configurar o Prisma com `url` (pooler) e `directUrl` (direct).
- [ ] Sei conectar aplicações Python com `psycopg`.
- [ ] Ativei a extensão `vector` e executei uma consulta de similaridade vetorial.

---
> **Navegação**: [⬅️ Aula Anterior: Escala e PITR](./04-escala-e-alta-disponibilidade.md) | [Módulo 05](./README.md) | [Módulo 06: Projetos Práticos ➡️](../06-projetos-praticos-e-desafios/README.md)
