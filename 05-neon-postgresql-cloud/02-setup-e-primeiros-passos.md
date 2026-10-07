# ⚡ Módulo 05: Neon Serverless PostgreSQL
## 📑 Aula 02: Setup do Projeto, Neon CLI e Conexões Diretas vs Pooling

> **Navegação**: [⬅️ Aula Anterior: O que é Neon](./01-introducao-ao-neon.md) | [Módulo 05](./README.md) | [Próxima Aula: Branching de Banco ➡️](./03-branching-de-banco-de-dados.md)

---

### 🎯 Objetivos de Aprendizagem
Ao final desta aula, você será capaz de:
- Criar e configurar um projeto gratuito no [Neon Console](https://console.neon.tech).
- Instalar e autenticar a ferramenta de linha de comando oficial **Neon CLI (`neonctl`)**.
- Compreender a diferença crítica entre o endpoint **Direto (*Direct*)** e o endpoint com **Pooler (*Pooled*)**.
- Conectar-se ao Neon via `psql`, DBeaver e scripts de aplicação.

---

### 1. Criando seu Primeiro Projeto no Neon Console

1. Acesse **[console.neon.tech](https://console.neon.tech)** e faça login com sua conta do GitHub ou Google.
2. Clique no botão **`New Project`**.
3. Preencha as configurações iniciais:
   - **Project Name**: Ex.: `curso-postgresql`
   - **Postgres Version**: Selecione a versão estável mais recente (PostgreSQL 16 ou 17).
   - **Region**: Selecione a região mais próxima da sua infraestrutura ou alunos (ex.: `US East (Ohio)` ou América do Sul se disponível).
4. Clique em **Create Project**. Em menos de 3 segundos, seu cluster estará pronto!

```mermaid
flowchart LR
    Browser[Navegador / Neon Console] -->|1 Clique| MicroVM[MicroVM Compute Inicializada]
    MicroVM -->|Storage Provisionado no Pageserver| Ready[(Cluster PostgreSQL 16 Ativo!)]
```

> [!IMPORTANT]
> Ao criar o projeto, o Neon exibirá a senha gerada e a **Connection String**. Copie e salve essa informação em um local seguro (ou gerenciador de segredos/.env).

---

### 2. A Ferramenta de Linha de Comando: `neonctl`

Para desenvolvedores e automações de CI/CD, o Neon disponibiliza o `neonctl`, permitindo gerenciar branches, projetos e bancos sem abrir o navegador.

#### Instalação via NPM:
```bash
npm install -g neonctl
```

#### Autenticação:
```bash
neonctl auth
```
*(Um link abrirá no seu navegador confirmando o login com sua conta do Neon).*

#### Comandos Úteis do `neonctl`:

```bash
# Listar todos os seus projetos
neonctl projects list

# Listar os branches do projeto ativo
neonctl branches list

# Obter a connection string diretamente no terminal
neonctl connection-string

# Executar uma query SQL rápida direto pelo terminal
neonctl sql "SELECT version();"
```

---

### 3. Anatomia dos Endpoints: Direct vs Pooled

Esta é uma das configurações mais importantes que todo engenheiro precisa dominar ao usar o Neon:

```mermaid
flowchart TD
    App[Sua Aplicação / Código] --> Escolha{Qual a natureza da aplicação?}

    Escolha -- ORM / Migrações / Servidor Dedicado Node/Java --> Direct["Endpoint Direto (Sem '-pooler')<br>Ex: ep-cool-lake-123456.us-east-2.aws.neon.tech"]
    Escolha -- Serverless / Lambdas / Next.js / Edge Workers --> Pooled["Endpoint com Pooler (Com '-pooler')<br>Ex: ep-cool-lake-123456-pooler.us-east-2.aws.neon.tech"]

    Direct --> Engine[Postgres Compute Direto]
    Pooled --> PgBouncer[PgBouncer Serverless Integrado]
    PgBouncer --> Engine
```

#### A. Endpoint Direto (`ep-xxxx.neon.tech`):
* Conecta-se diretamente ao backend do Postgres.
* **Obrigatório para**:
  - Comandos DDL que executam migrações de schema (ex.: `prisma migrate`, `liquibase`, `flyway`).
  - Queries que usam `LISTEN` e `NOTIFY`.
  - Tabelas temporárias ou instruções preparadas nomeadas a nível de sessão.

#### B. Endpoint com Pooler (`ep-xxxx-pooler.neon.tech`):
* Conecta-se a um proxy **PgBouncer** integrado configurado em modo de transação (*Transaction Pooling*).
* **Obrigatório para**:
  - Aplicações Serverless (Next.js App Router, Vercel Functions, AWS Lambda).
  - Ambientes onde centenas de funções efêmeras abrem e fecham conexões em rajadas.
  - Evita o esgotamento do limite de conexões (`sorry, too many clients already`).

---

### 4. Conectando-se ao Neon via `psql`

Você pode conectar o cliente `psql` instalado na sua máquina diretamente ao Neon:

```bash
psql "postgresql://alex:AbCdEfGh123@ep-divine-pond-123456.us-east-2.aws.neon.tech/neondb?sslmode=require"
```

> [!CAUTION]
> No Neon, a flag **`?sslmode=require`** é estritamente obrigatória por questões de segurança de tráfego de dados na internet. Conexões sem SSL criptografado serão rejeitadas.

---

### 5. Executando no SQL Editor Embutido do Neon

Se um aluno estiver em um computador bloqueado ou sem Docker/psql instalado:
1. No painel do Neon, clique na aba **`SQL Editor`** no menu lateral.
2. Digite sua query e clique no botão **`Run`** (ou use o atalho `Ctrl + Enter` / `Cmd + Enter`).
3. O resultado e o tempo de resposta aparecerão imediatamente na tela.

```sql
-- Teste de conectividade e extensões no Neon:
SELECT 
    current_database() AS banco_atual,
    current_user AS usuario_atual,
    inet_server_addr() AS ip_servidor,
    version();
```

---

### 📝 Checklist de Conclusão da Aula

- [ ] Criei minha conta gratuita e inicializei meu projeto no Neon.
- [ ] Entendi por que o endpoint com `-pooler` é necessário em Serverless.
- [ ] Executei meu primeiro comando no SQL Editor do console do Neon.
- [ ] Sei que a conexão em nuvem exige o parâmetro `sslmode=require`.

---
> **Navegação**: [⬅️ Aula Anterior: O que é Neon](./01-introducao-ao-neon.md) | [Módulo 05](./README.md) | [Próxima Aula: Branching de Banco ➡️](./03-branching-de-banco-de-dados.md)
