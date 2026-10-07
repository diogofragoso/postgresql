# ⚡ Módulo 05: Neon Serverless PostgreSQL
## 📑 Aula 03: Database Branching: Git para Bancos de Dados

> **Navegação**: [⬅️ Aula Anterior: Setup e Conexões](./02-setup-e-primeiros-passos.md) | [Módulo 05](./README.md) | [Próxima Aula: Escala e PITR ➡️](./04-escala-e-alta-disponibilidade.md)

---

### 🎯 Objetivos de Aprendizagem
Ao final desta aula, você será capaz de:
- Compreender o conceito inovador de **Database Branching** e a tecnologia *Copy-on-Write*.
- Criar branches instantâneos isolados com dados e schema ou apenas schema.
- Integrar branches do banco com fluxos de desenvolvimento Git, Pull Requests e CI/CD.
- Aplicar o branching em sala de aula para fornecer um banco de dados independente e idêntico para cada estudante.
- Gerenciar branches pelo Neon Console e pela CLI (`neonctl`).

---

### 1. O que é Database Branching?

No desenvolvimento moderno de software com Git, nós criamos branches para desenvolver novas funcionalidades de código de forma isolada:
```bash
git checkout -b feature/novo-checkout
```

Por décadas, o banco de dados foi o **elo fraco** desse fluxo: desenvolvedores precisavam usar bancos locais desatualizados ou competir por um ambiente de "staging" compartilhado onde um desenvolvedor quebrava os dados do outro.

O **Neon** resolveu esse problema histórico ao trazer branches nativos para o PostgreSQL em nível de armazenamento:

```mermaid
gitGraph
    commit id: "Initial Schema"
    commit id: "Carga Inicial de Dados"
    branch feature/nova-tabela
    checkout feature/nova-tabela
    commit id: "ALTER TABLE adiciona coluna"
    commit id: "Testa carga de 50.000 clientes"
    checkout main
    commit id: "Produção segue operando 100% isolada"
```

---

### 2. Como Funciona Fisicamente: A Mágica do *Copy-on-Write*

Como é possível criar um branch de um banco de dados de 500 GB em apenas **1 segundo** sem custo adicional exorbitante?

```mermaid
flowchart TD
    subgraph Parent["Branch Principal (main)"]
        P_Page1["Página A (v1)"]
        P_Page2["Página B (v1)"]
        P_Page3["Página C (v1)"]
    end

    subgraph BranchDev["Novo Branch (dev-aluno-1)"]
        D_Delta["Página B modificada (v2) [Delta Exclusivo do Aluno]"]
    end

    Parent -.->|Leitura Compartilhada de Páginas Não Modificadas| BranchDev
    BranchDev -->|Grava apenas novidades| D_Delta
```

1. **Sem cópia inicial de bytes**: Ao criar o branch, o Pageserver do Neon cria apenas um ponteiro de metadados apontando para o instante exato do branch pai (Log Sequence Number - LSN).
2. **Copy-on-Write**: Enquanto você apenas lê dados no branch filho, você lê as páginas compartilhadas do pai.
3. **Isolamento de Escrita**: Quando você executa um `INSERT`, `UPDATE` ou `DROP TABLE` no branch filho, apenas os novos blocos modificados (*deltas*) são gravados para aquele branch específico. O branch pai permanece completamente inalterado e seguro!

> [!NOTE]
> Você pode rodar `DROP TABLE clientes;` dentro de uma branch de teste sem nenhum medo: a tabela no branch `main` continuará 100% intacta!

---

### 3. Tipos de Branches no Neon

Ao criar um branch, você escolhe entre duas estratégias:

| Tipo de Branch | O que Contém? | Melhor Caso de Uso |
| :--- | :--- | :--- |
| **Branch Completo (Schema + Data)** | Schema completo e todos os dados do branch pai no momento exato da criação | Debugar bugs reais de produção, testes de carga, ambiente de homologação |
| **Branch Apenas Estrutura (Schema Only)** | Todas as tabelas, tipos e views, mas **zerada de dados** | Ambientes de testes unitários limpos, conformidade rígida com LGPD/GDPR |

---

### 4. Criando Branches na Prática

#### A. Pelo Console Web do Neon:
1. No menu lateral do projeto, clique em **`Branches`**.
2. Clique no botão **`New Branch`**.
3. Defina o nome: `feature-migracao-v2`.
4. Escolha o branch pai: `main`.
5. Clique em **Create Branch**.
6. Pronto! O Neon fornecerá uma nova Connection String exclusiva para este branch.

#### B. Pela CLI (`neonctl`):
```bash
# Criar um branch completo a partir da main
neonctl branches create --name feature-migracao-v2

# Criar um branch apenas com o schema (sem dados sensíveis)
neonctl branches create --name teste-limpo-schema --schema-only

# Obter a string de conexão do novo branch
neonctl connection-string --branch feature-migracao-v2
```

---

### 5. Cenário Pedagógico: Um Banco por Aluno sem Conflitos

Em aulas tradicionais de laboratório, se 30 alunos usam o mesmo banco de dados:
* O aluno A apaga os registros que o aluno B estava consultando.
* O aluno C altera a estrutura da tabela e quebra a query do aluno D.

#### O Fluxo Recomendado para Professores:
1. O professor cria a branch `template-aula-05` com os dados e tabelas preparadas para a atividade.
2. Cada estudante cria seu branch a partir do template do professor:
   - `aluno-joao`
   - `aluna-maria`
   - `aluno-pedro`
3. Cada estudante recebe sua própria URI de conexão, com total autonomia para testar, errar, dropar tabelas e refazer.
4. Ao final da aula, o professor pode inspecionar o branch de cada aluno ou simplesmente deletá-los em lote!

```mermaid
flowchart TD
    Prof["Branch do Professor: template-aula"] --> B1["Branch Aluno: joao"]
    Prof --> B2["Branch Aluna: maria"]
    Prof --> B3["Branch Aluno: pedro"]
    
    B1 -->|Trabalho Prático Independente| V1[(Banco Isolado 1)]
    B2 -->|Trabalho Prático Independente| V2[(Banco Isolado 2)]
    B3 -->|Trabalho Prático Independente| V3[(Banco Isolado 3)]
```

---

### 6. Branches Efêmeros em Pipelines de CI/CD (GitHub Actions)

No ecossistema corporativo, branches do Neon são utilizados em Pull Requests:

```mermaid
sequenceDiagram
    actor Dev as Desenvolvedor
    participant GH as GitHub Pull Request
    participant Neon as Neon API
    participant Test as Testes de Integração

    Dev->>GH: Abre Pull Request #42
    GH->>Neon: Cria branch efêmero 'pr-42'
    Neon-->>GH: Retorna Connection String do branch 'pr-42'
    GH->>Test: Roda migrações e testes no branch temporário
    Test-->>GH: Testes passaram com 100% de sucesso!
    Dev->>GH: Faz Merge do PR na main
    GH->>Neon: Deleta automaticamente o branch 'pr-42'
```

---

### 7. Atividade Prática 06: Laboratório de Branching e Destruição Segura

Neste laboratório, você experimentará a maior vantagem competitiva do Neon: o poder de testar operações destrutivas ou migrações complexas em uma cópia exata de produção sem correr nenhum risco de corromper os dados reais.

#### Roteiro Passo a Passo:

**Passo 1 (No Branch `main`)**: Crie uma tabela com dados importantes:
```sql
CREATE TABLE clientes_producao (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome TEXT NOT NULL,
    saldo NUMERIC(10,2) NOT NULL
);

INSERT INTO clientes_producao (nome, saldo) VALUES
    ('Empresa Alfa', 50000.00),
    ('Empresa Beta', 120000.00);
```

**Passo 2**: Crie um branch chamado `experimento-destrutivo` a partir do `main`:
- No Neon Console: clique em **Branches -> New Branch** -> Nome: `experimento-destrutivo`.
- Ou via CLI: `neonctl branches create --name experimento-destrutivo`

**Passo 3 (No Branch `experimento-destrutivo`)**:
Conecte-se ao novo branch (via Beekeeper Studio ou Neon SQL Editor) e execute o pior pesadelo de um banco tradicional:
```sql
-- Dropar a tabela inteira!
DROP TABLE clientes_producao;

-- Verificar que a tabela sumiu no branch de teste:
-- SELECT * FROM clientes_producao; -> ERRO: relation does not exist
```

**Passo 4 (De volta ao Branch `main`)**:
Reconecte-se ao branch principal e execute:
```sql
SELECT * FROM clientes_producao;
```

<details>
<summary>💡 Clique para ver o resultado e explicação técnica</summary>

**Resultado:**
Todos os registros de `Empresa Alfa` e `Empresa Beta` continuam **100% intactos e preservados** no branch `main`!

Graças à tecnologia de *Copy-on-Write* do Pageserver do Neon:
1. O comando `DROP TABLE` registrou a exclusão apenas nos metadados exclusivos do branch `experimento-destrutivo`.
2. O branch `main` nunca foi tocado nem sofreu locks de tabela.
3. Agora você pode simplesmente apagar o branch de teste no painel do Neon (`neonctl branches delete experimento-destrutivo`), sem deixar nenhum rastro ou custo residual!
</details>

---
> **Navegação**: [⬅️ Aula Anterior: Setup e Conexões](./02-setup-e-primeiros-passos.md) | [Módulo 05](./README.md) | [Próxima Aula: Escala e PITR ➡️](./04-escala-e-alta-disponibilidade.md)
