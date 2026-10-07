# 🐘 Módulo 01: Fundamentos de Banco de Dados e PostgreSQL
## 📑 Aula 02: Instalação, Docker e Ferramental de Trabalho

> **Navegação**: [⬅️ Aula Anterior: Introdução](./01-introducao-ao-banco-de-dados.md) | [Módulo 01](./README.md) | [Próxima Aula: Arquitetura Interna ➡️](./03-arquitetura-postgresql.md)

---

### 🎯 Objetivos de Aprendizagem
Ao final desta aula, você será capaz de:
- Subir uma instância do PostgreSQL 16 utilizando **Docker** e **Docker Compose**.
- Conectar-se ao banco via terminal utilizando o cliente interativo **`psql`**.
- Dominar os meta-comandos mais frequentes do `psql` (`\l`, `\c`, `\dt`, `\d+`, `\dn`, `\q`).
- Configurar interfaces gráficas como **DBeaver** ou **pgAdmin**.
- Entender a estrutura padrão de uma Connection String (URI).

---

### 1. Métodos de Instalação: Qual Escolher?

Existem três maneiras principais de rodar o PostgreSQL durante os estudos e desenvolvimento:

```mermaid
graph TD
    Opcao[Como rodar o PostgreSQL?] --> InstalaLocal[Instalador Nativo SO]
    Opcao --> ContainerDocker[Container Docker / Compose]
    Opcao --> CloudNeon[Nuvem Serverless: Neon]

    InstalaLocal -->|Prós: Serviço nativo<br>Contras: Polui o SO, conflitos de porta| IL[Linux / Windows / macOS]
    ContainerDocker -->|Prós: Isolado, reproduzível, descartável<br>Padrão na indústria| CD[Docker Desktop / Podman]
    CloudNeon -->|Prós: Sem instalar nada localmente, branches instantâneos| CN[Neon Console Grátis]
```

> [!TIP]
> Para o aprendizado moderno, recomendamos o **Docker** para ambiente offline/local e o **Neon PostgreSQL** para trabalhos em nuvem e projetos em equipe.

---

### 2. Subindo com Docker e Docker Compose (Recomendado)

O Docker garante que toda a turma de alunos execute exatamente a mesma versão do PostgreSQL, com as mesmas configurações de porta, senhas e volumes, independentemente de estarem no Windows, Linux ou macOS.

#### Arquivo `compose.yaml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres_estudos
    restart: always
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secretpassword123
      POSTGRES_DB: universidade
    ports:
      - "5432:5432"
    volumes:
      - postgres_dados:/var/lib/postgresql/data

volumes:
  postgres_dados:
    driver: local
```

#### Comandos de gerenciamento:

```bash
# Iniciar o container em segundo plano (detached mode)
docker compose up -d

# Verificar se o container está rodando e saudável
docker compose ps

# Visualizar logs em tempo real
docker compose logs -f postgres

# Parar o container mantendo os dados salvos no volume
docker compose down
```

> [!WARNING]
> A porta padrão do PostgreSQL é a **5432**. Se você já tiver outra instância do Postgres instalada na máquina, haverá conflito de porta (`address already in use`). Nesse caso, altere o mapeamento no compose para `"5433:5432"`.

---

### 3. Anatomia de uma Connection String (URI)

Tanto aplicações em Node.js, Python, Java quanto ferramentas gráficas usam a especificação de URI de conexão:

```text
postgresql://[usuario]:[senha]@[host]:[porta]/[nome_do_banco]?sslmode=[modo]
```

#### Exemplo prático:
```text
postgresql://admin:secretpassword123@localhost:5432/universidade?sslmode=disable
```

* **Protocolo**: `postgresql://` (ou `postgres://`)
* **Usuário**: `admin`
* **Senha**: `secretpassword123`
* **Host**: `localhost` (ou `127.0.0.1`, ou endpoint na nuvem)
* **Porta**: `5432`
* **Database**: `universidade`
* **Parâmetros**: `sslmode=disable` (para local) ou `sslmode=require` (para serviços em nuvem como o Neon).

---

### 4. Dominando o Terminal: Cliente Interativo `psql`

O `psql` é a ferramenta de linha de comando mais poderosa e rápida para administração do PostgreSQL.

#### Conectando via Docker:
```bash
docker exec -it postgres_estudos psql -U admin -d universidade
```

#### Conectando via terminal local:
```bash
psql -h localhost -p 5432 -U admin -d universidade
```

#### Meta-comandos Essenciais (Começam com barra invertida `\`):

| Comando | Descrição | Equivalente em outros bancos |
| :--- | :--- | :--- |
| `\l` | Lista todos os bancos de dados do servidor | `SHOW DATABASES;` |
| `\c nome_banco` | Conecta-se a outro banco de dados | `USE nome_banco;` |
| `\dt` | Lista todas as tabelas do schema atual | `SHOW TABLES;` |
| `\dt+` | Lista tabelas com tamanho em disco e descrição | Detalhes físicos |
| `\d nome_tabela` | Descreve colunas, tipos e constraints da tabela | `DESCRIBE nome_tabela;` |
| `\dn` | Lista todos os Schemas existentes | N/A |
| `\du` | Lista usuários e suas permissões/roles | `SELECT * FROM mysql.user;` |
| `\x` | Alterna modo de exibição expandido (ótimo para linhas com muitas colunas) | Formatação vertical |
| `\timing` | Liga/Desliga o cronômetro de tempo de execução das queries | Benchmark rápido |
| `\i caminho/arquivo.sql` | Executa um script SQL a partir do arquivo | Executar script externo |
| `\q` | Sai do cliente psql | Sair / Exit |

> [!NOTE]
> Comandos SQL tradicionais exigem ponto e vírgula no final (`;`). Meta-comandos do `psql` (que iniciam com `\`) **NÃO** utilizam ponto e vírgula.

---

### 5. Interfaces Gráficas Recomendadas (GUIs)

Para desenvolvimento diário, interfaces visuais aumentam a produtividade:

1. **DBeaver Community**: Software livre, multiplataforma, suporta diagramas ER automáticos e autocomplete avançado.
2. **pgAdmin 4**: Ferramenta oficial mantida pelo PostgreSQL Group.
3. **TablePlus**: Interface minimalista, extremamente leve e nativa.
4. **Extensão do VS Code (Database Client / SQLTools)**: Permite rodar queries diretamente no seu editor de código.

---

### 6. Exercício Prático: O Primeiro "Hello World" Relacional

Conecte-se ao seu PostgreSQL via `psql` ou terminal e execute as instruções abaixo:

```sql
-- 1. Verificar a versão exata do PostgreSQL
SELECT version();

-- 2. Criar uma tabela simples de teste
CREATE TABLE boas_vindas (
    id SERIAL PRIMARY KEY,
    mensagem VARCHAR(100) NOT NULL,
    criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. Inserir o primeiro registro
INSERT INTO boas_vindas (mensagem)
VALUES ('Olá, PostgreSQL 16! Bem-vindo ao curso.');

-- 4. Consultar os dados
SELECT * FROM boas_vindas;
```

<details>
<summary>👁️ Clique aqui para ver o resultado esperado</summary>

```text
 id |                   mensagem                   |         criado_em          
----+----------------------------------------------+----------------------------
  1 | Olá, PostgreSQL 16! Bem-vindo ao curso.      | 2026-10-07 14:45:00.123456
(1 row)
```
</details>

---

### 📝 Checklist de Conclusão da Aula

- [ ] Instalei ou subi o container Docker do PostgreSQL com sucesso.
- [ ] Consegui conectar usando o `psql` e executei o comando `\l`.
- [ ] Compreendi os componentes da Connection String `postgresql://user:pass@host:port/db`.
- [ ] Criei a tabela de teste e confirmei a gravação do registro.

---
> **Navegação**: [⬅️ Aula Anterior: Introdução](./01-introducao-ao-banco-de-dados.md) | [Módulo 01](./README.md) | [Próxima Aula: Arquitetura Interna ➡️](./03-arquitetura-postgresql.md)
