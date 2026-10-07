# 🐘 Módulo 01: Fundamentos de Banco de Dados e PostgreSQL
## 📑 Trilha B: Setup com Neon Serverless Cloud e Beekeeper Studio Portable

> **Navegação**: [⬅️ Voltar ao Portal de Escolha de Trilha](./02-instalacao-e-configuracao.md) | [Módulo 01](./README.md) | [Avançar para a Aula 03: Arquitetura Interna ➡️](./03-arquitetura-postgresql.md)

---

### 🎯 Perfil Desta Trilha
* **Público-Alvo**: Turmas de Desenvolvimento de Software (Frontend, Backend, Mobile, Fullstack), Ciência de Dados, Análise de Sistemas ou turmas com tempo reduzido de aula onde o foco é **aprender SQL diretamente sem gerenciar servidores Linux**.
* **Pré-requisitos**: Apenas um navegador web moderno e acesso à internet. Funciona em qualquer sistema operacional (Windows, macOS, Linux, ChromeOS).
* **Objetivo**: Estar com um banco PostgreSQL 16 de produção ativo e conectado em **menos de 3 minutos**, utilizando o plano gratuito do Neon e o cliente portátil **Beekeeper Studio Portable** (ou o SQL Editor Web integrado).

> [!TIP]
> **Professor**: Esta trilha elimina completamente dores de cabeça com instalação de Docker, configurações de portas e problemas de máquina em sala de aula. Se sua intenção for ensinar comandos SQL, modelagem e queries de forma rápida, **esta é a trilha recomendada para sua turma**!

---

### 🗺️ Fluxo de Trabalho da Trilha B

```mermaid
flowchart TD
    A["1. Criar Conta Gratuita no Neon Console (via GitHub/Google)"] --> B["2. Criar Novo Projeto PostgreSQL 16 (Leva 3 Segundos!)"]
    B --> C["3. Copiar a Connection String Gerada"]
    C --> D{"4. Escolha do Cliente pelo Estudante"}
    D -- Opção 1: Zero Instalação --> E["Usar o SQL Editor no Próprio Navegador"]
    D -- Opção 2: Interface Desktop --> F["Beekeeper Studio Portable (Roda sem Admin)"]
    E --> G["5. Executar o Teste de Validação"]
    F --> G
    G --> H["Pronto para as Aulas de SQL!"]
```

---

### 1. Criando seu Banco de Dados no Neon Console (Nuvem Gratuita)

O Neon disponibiliza um plano gratuito perpétuo voltado para estudos e protótipos, que não exige dados de cartão de crédito.

#### Passo 1.1: Acessar e Autenticar
1. Acesse **[console.neon.tech](https://console.neon.tech)** no seu navegador.
2. Faça login com sua conta do **GitHub** ou **Google**.

#### Passo 1.2: Criar o Projeto do Curso
1. No painel principal, clique no botão **`New Project`**.
2. Preencha as configurações:
   - **Project Name**: Exemplo: `lab-estudos-postgres`
   - **Postgres Version**: Selecione a versão padrão mais recente (PostgreSQL 16 ou 17).
   - **Region**: Escolha a região mais próxima (ex.: `US East (Ohio)` ou similar).
3. Clique em **Create Project**. Em apenas 3 segundos, seu cluster estará provisionado e pronto para receber queries!

#### Passo 1.3: Copiar a Connection String
Assim que o projeto for criado, o Neon exibirá a sua **Connection String** no formato:
```text
postgresql://alex:AbCdEfGh123@ep-divine-pond-123456.us-east-2.aws.neon.tech/neondb?sslmode=require
```

> [!IMPORTANT]
> Copie e salve essa URL de conexão! Ela contém o seu usuário, senha, endereço do servidor em nuvem e a flag de segurança obrigatória `sslmode=require`.

---

### 2. Opção A: Executar Diretamente no Navegador (Web SQL Editor)

Se o aluno estiver utilizando um computador com bloqueio rígido, um Chromebook ou não quiser baixar nenhum arquivo:

1. No menu lateral do console do Neon, clique na aba **`SQL Editor`**.
2. Uma janela completa de edição de SQL se abrirá.
3. Digite suas consultas e pressione `Ctrl + Enter` (ou clique no botão verde **Run**).
4. O resultado da query e o tempo exato de resposta em milissegundos aparecerão na tela.

---

### 3. Opção B: Beekeeper Studio Portable (Cliente Gráfico Desktop)

Para estudantes que desejam o conforto de um aplicativo desktop para ver tabelas, ordenar dados e ter histórico de queries, o **Beekeeper Studio Community Edition (Portable)** é a solução perfeita.

#### 🐝 Por que o Beekeeper Studio Portable?
* **Zero Instalação**: O executável é 100% portátil. Ele abre com duplo clique sem precisar de instalador e **sem exigir permissões de administrador no computador**.
* **Pode ser levado em um Pen Drive**: O aluno pode salvar o aplicativo e levá-lo para as aulas práticas.
* **Importação Instantânea**: Não precisa preencher host, porta ou SSL manualmente; basta colar a Connection String do Neon!

#### Como Baixar a Versão Portátil (Open Source):
Acesse a página oficial de lançamentos: [Releases do Beekeeper Studio no GitHub](https://github.com/beekeeper-studio/beekeeper-studio/releases) (ou [beekeeperstudio.io](https://www.beekeeperstudio.io)):

* **Windows**: Baixe o arquivo executável portátil `Beekeeper-Studio-Portable-x.x.x.exe`.
* **Linux Desktop**: Baixe o executável `Beekeeper-Studio-x.x.x.AppImage` (clique com botão direito -> Propriedades -> Permitir executar, ou rode `chmod +x` no terminal).
* **macOS**: Baixe o instalador `.dmg` (compatível com Apple Silicon M1/M2/M3 e Intel).

#### Como Conectar ao Neon com o Beekeeper Studio em 3 Cliques:

1. Abra o **Beekeeper Studio** com duplo clique.
2. Na tela inicial de conexões, clique no botão **"Import from URL"** no topo:
   
   ```mermaid
   flowchart LR
       Neon[Copie a URL do Neon Console] --> ImportBtn["Clique em 'Import from URL' no Beekeeper"]
       ImportBtn --> Paste[Cole a URL Completa]
       Paste --> AutoConfig["O Beekeeper preenche Host, Usuário e SSL Mode: require automaticamente!"]
       AutoConfig --> Connected[(Conexão Estabelecida com Sucesso!)]
   ```

3. Cole a Connection String do Neon copiada no Passo 1.3:
   ```text
   postgresql://alex:AbCdEfGh123@ep-divine-pond-123456.us-east-2.aws.neon.tech/neondb?sslmode=require
   ```
4. Observe que o Beekeeper preencherá automaticamente todos os campos e marcará a caixa **SSL Mode: require**.
5. Clique no botão **Test Connection**. Ao receber a notificação verde de sucesso, clique em **Salvar e Conectar**.
6. Dê um nome amigável para a conexão, como *"Neon Nuvem Aula"*.

---

### 4. Exercício Prático: Validação do Ambiente na Nuvem

Seja através do SQL Editor no navegador ou do Beekeeper Studio Desktop, execute as instruções abaixo para validar que seu banco na nuvem está 100% funcional:

```sql
-- 1. Verificar a versão exata do PostgreSQL na nuvem
SELECT version();

-- 2. Inspecionar o banco atual e usuário conectado
SELECT 
    current_database() AS banco_conectado,
    current_user AS usuario_conectado,
    inet_server_addr() AS ip_do_servidor_cloud;

-- 3. Criar uma tabela simples de teste
CREATE TABLE boas_vindas_cloud (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mensagem TEXT NOT NULL,
    criado_em TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 4. Inserir o primeiro registro
INSERT INTO boas_vindas_cloud (mensagem)
VALUES ('Ambiente Neon Serverless configurado com sucesso para o curso!');

-- 5. Consultar os dados
SELECT * FROM boas_vindas_cloud;
```

<details>
<summary>👁️ Clique aqui para ver o resultado esperado</summary>

```text
 id |                           mensagem                            |           criado_em           
----+---------------------------------------------------------------+-------------------------------
  1 | Ambiente Neon Serverless configurado com sucesso para o curso! | 2026-10-07 15:30:00.654321-03
(1 row)
```
</details>

---

### 💡 Vantagem Extra para Estudantes: Database Branching

Como você está no Neon, você já tem acesso ao recurso de **Branches instantâneos** (estudaremos a fundo no Módulo 05). 

Se você for fazer uma tarefa de casa ou quiser testar comandos perigosos como `DELETE` ou `DROP TABLE` sem medo:
1. No painel do Neon, clique em **Branches -> New Branch**.
2. Crie uma branch chamada `minha-tarefa`.
3. Você ganha uma nova Connection String isolada para experimentar livremente sem medo de quebrar seu banco principal!

---

### 📝 Checklist de Conclusão da Trilha B

- [ ] Criei minha conta gratuita no Neon Console.
- [ ] Provisionei meu projeto PostgreSQL 16 na nuvem em menos de 1 minuto.
- [ ] Copiei e guardei minha Connection String com segurança.
- [ ] Executei uma query no SQL Editor do navegador ou conectei via Beekeeper Studio Portable.
- [ ] Executei o script de teste e validei a criação da tabela `boas_vindas_cloud`.

---
> **Navegação**: [⬅️ Voltar ao Portal de Escolha de Trilha](./02-instalacao-e-configuracao.md) | [Módulo 01](./README.md) | [Avançar para a Aula 03: Arquitetura Interna ➡️](./03-arquitetura-postgresql.md)
