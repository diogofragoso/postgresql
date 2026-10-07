# ⚡ Módulo 05: Neon Serverless PostgreSQL

<div align="center">

![Nível](https://img.shields.io/badge/Nível-Cloud_Native_e_DevOps-green?style=for-the-badge)
![Aulas](https://img.shields.io/badge/Aulas-5_Capítulos-blue?style=for-the-badge)
![Foco](https://img.shields.io/badge/Foco-Serverless_PostgreSQL_e_Branching-teal?style=for-the-badge)

</div>

Neste módulo, você explora a revolução do banco de dados relacional serverless com o **Neon**, aprendendo como a separação entre Computação e Armazenamento transforma os fluxos de trabalho de desenvolvimento, testes, CI/CD e produção.

---

## 🗺️ Mapa de Conteúdo do Módulo

```mermaid
flowchart LR
    A["01. Arquitetura Desacoplada"] --> B["02. Setup e Conexões Direct/Pooler"]
    B --> C["03. Database Branching (Git-like)"]
    C --> D["04. Autoscaling e PITR"]
    D --> E["05. Integração Moderna e IA (pgvector)"]
```

---

## 📑 Aulas Disponíveis

1. [**Aula 01: O que é o Neon e a Revolução da Separação entre Compute e Storage**](./01-introducao-ao-neon.md)
   - O problema da nuvem tradicional acoplada (RDS/VPS).
   - A trinca do Neon: Compute MicroVMs, Safekeepers e Pageservers.
   - Por que o Neon mantém 100% de compatibilidade com o PostgreSQL upstream.
   - Economia drástica com Scale to Zero.

2. [**Aula 02: Setup do Projeto, Neon CLI e Conexões Diretas vs Pooling**](./02-setup-e-primeiros-passos.md)
   - Criando projetos gratuitos no Neon Console.
   - Gerenciamento via terminal com `neonctl`.
   - Direct Endpoint vs Pooled Endpoint (PgBouncer integrado).
   - Requisito de criptografia com `sslmode=require`.
   - Utilizando o SQL Editor embutido no navegador.

3. [**Aula 03: Database Branching: Git para Bancos de Dados**](./03-branching-de-banco-de-dados.md)
   - O que é Database Branching e a mágica do *Copy-on-Write*.
   - Branches completos vs Branches apenas com schema.
   - Estratégia de sala de aula: Um banco isolado para cada estudante sem conflito.
   - Branches efêmeros em pipelines de CI/CD (GitHub Actions).

4. [**Aula 04: Autoscaling, Scale to Zero e Point-in-Time Recovery (PITR)**](./04-escala-e-alta-disponibilidade.md)
   - Autoscaling vertical a quente sem reiniciar o servidor.
   - Como funciona a suspensão e o cold start (< 500ms).
   - Point-in-Time Recovery (PITR): Como restaurar desastres em 2 segundos voltando no tempo.

5. [**Aula 05: Integração com Aplicações Modernas (Node.js, Prisma, Drizzle, Python e IA)**](./05-integracao-com-aplicacoes.md)
   - O driver `@neondatabase/serverless` sobre HTTP/WebSockets.
   - Configurando Prisma ORM com `directUrl` e connection pooling.
   - Conexão tipada com Drizzle ORM.
   - Conexão em Python com `psycopg`.
   - Busca semântica e IA vetorial com a extensão `pgvector`.

---

## 🧭 Navegação Rápida
* [⬅️ Voltar para o Módulo 04: Recursos Avançados SQL](../04-recursos-avancados-sql/README.md)
* [Ir para o Módulo 06: Projetos Práticos e Desafios ➡️](../06-projetos-praticos-e-desafios/README.md)
