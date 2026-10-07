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

### 5. Interfaces Gráficas: Beekeeper Studio Portable (Recomendado para Aulas)

Para desenvolvimento diário e visualização de tabelas, uma interface gráfica intuitiva acelera o aprendizado dos estudantes.

#### 🐝 A Escolha Ideal para Laboratórios: Beekeeper Studio Community (Portable)

Em laboratórios de faculdades, escolas técnicas ou computadores corporativos, estudantes frequentemente enfrentam **bloqueios de permissão de administrador** que impedem instalar programas convencionais.

O **Beekeeper Studio Community Edition (Versão Portable)** resolve esse problema por completo:

```mermaid
flowchart LR
    Download[Download do Arquivo Portátil] --> Pendrive["Executa Direto da Pasta ou Pen Drive<br>(Sem precisar de Administrador!)"]
    Pendrive --> Conexao["Conecta em 1 Clique:<br>Docker Local ou Neon Cloud"]
    Conexao --> Pratica[Pronto para as Aulas Práticas!]
```

> [!TIP]
> **Por que recomendamos o Beekeeper Studio para os estudantes?**
> * **Zero Instalação**: O executável roda diretamente com duplo clique (não altera registros do Windows nem precisa de privilégios de `root`/administrador).
> * **Interface Limpa e Focada**: Diferente de ferramentas pesadas e com excesso de opções complexas (como pgAdmin ou DBeaver), o Beekeeper possui foco total na escrita e execução de SQL.
> * **Importação Instantânea via URL**: Possui o botão **"Import from URL"**, permitindo colar a Connection String inteira do Neon ou do Docker sem preencher campos manualmente.
> * **Visualizador e Editor de Dados**: Permite filtrar, ordenar e editar registros em formato de planilha visual interativa.
> * **Histórico Automático**: Guarda o histórico de todas as consultas executadas para fácil recuperação durante os exercícios.

#### Onde Baixar a Versão Portátil (Open Source):
Acesse a página oficial de lançamentos no GitHub: [Releases do Beekeeper Studio](https://github.com/beekeeper-studio/beekeeper-studio/releases) (ou [beekeeperstudio.io](https://www.beekeeperstudio.io)):

* **Windows**: Baixe o arquivo `Beekeeper-Studio-Portable-x.x.x.exe`.
* **Linux**: Baixe o arquivo `Beekeeper-Studio-x.x.x.AppImage` (basta torná-lo executável com `chmod +x` e abrir com duplo clique).
* **macOS**: Baixe o instalador `.dmg` compatível com Apple Silicon (arm64) ou Intel.

#### Como Configurar sua Primeira Conexão no Beekeeper:

1. Abra o executável do **Beekeeper Studio**.
2. Na tela inicial, clique no botão **"Import from URL"** (canto superior da tela de nova conexão).
3. **Para o PostgreSQL Local (Docker)**:
   - Cole a URI: `postgresql://admin:secretpassword123@localhost:5432/universidade?sslmode=disable`
4. **Para o Neon PostgreSQL (Nuvem)**:
   - Cole a URI obtida no painel do Neon: `postgresql://alex:senha@ep-divine-pond-123456.us-east-2.aws.neon.tech/neondb?sslmode=require`
5. Clique no botão **Test Connection** (Testar Conexão). Se aparecer a mensagem verde de sucesso, clique em **Connect** e salve com o nome *"Postgres Aula"*!

---

#### 🛠️ Outras Opções Disponíveis no Mercado

Caso o aluno já tenha preferência por outra ferramenta instalada:
1. **DBeaver Community**: Gratuito e poderoso, com suporte a diagramas ER automáticos (mais pesado em consumo de memória).
2. **pgAdmin 4**: Ferramenta oficial web/desktop mantida pelo PostgreSQL Global Development Group.
3. **TablePlus**: Interface minimalista e ultrarrápida (versão gratuita possui limitação de 2 abas abertas).
4. **Extensão Database Client (VS Code)**: Excelente para quem deseja rodar queries sem sair do editor de código.

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
