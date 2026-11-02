# ADR-0013: Usar Azure SQL Database como banco relacional, não PostgreSQL Flexible Server

**Status:** aceito
**Data:** 2026-11-02
**Decisores:** Arquitetura (Roberto Rivelino Barbosa)
**Substitui:** a escolha de banco do [ADR-0011](0011-migrar-plataforma-de-aws-eks-para-azure-aks.md) (Azure Database for PostgreSQL — Flexible Server). O modelo de dados do [ADR-0001](0001-multi-tenancy-schema-por-tenant.md) é ajustado, não revogado.

## Contexto

Desde o primeiro desenho, o banco do ArqFlow é PostgreSQL — no ADR-0011, Azure Database for PostgreSQL Flexible Server. A diretriz atual para implantação e operação é manter a **stack Microsoft**: o banco relacional deve ser o **Microsoft SQL**, na forma de serviço gerenciado. O ArqFlow guarda demandas, cenários, estimativas, metadados de diagramas e trilha de auditoria — um modelo relacional comum, sem extensões específicas de PostgreSQL (não usa PostGIS, `jsonb` como eixo do modelo nem extensões de busca).

## Alternativas consideradas

### Opção A — Manter Azure Database for PostgreSQL Flexible Server
- Prós: nenhum retrabalho; a fatia vertical (`arqflow-app`, [ADR-0008](0008-stack-de-implementacao-da-fatia-vertical-inicial.md)) já roda em PostgreSQL.
- Contras: fora da diretriz de stack Microsoft; DBAs e ferramentas corporativas (SSMS, Defender for SQL, auditoria do SQL) não se aplicam da mesma forma.

### Opção B — Azure SQL Managed Instance
- Prós: compatibilidade quase total com SQL Server on-premises (SQL Agent, cross-database queries, CLR).
- Contras: nenhuma dessas capacidades é usada pelo ArqFlow; custo e tempo de provisionamento bem maiores; pensada para migração lift-and-shift, não para aplicação nova.

### Opção C — Azure SQL Database (banco único, vCore General Purpose)
- Prós: PaaS do Microsoft SQL; alta disponibilidade e redundância de zona gerenciadas; **autenticação só por Microsoft Entra ID** (Managed Identity dos Container Apps, sem usuário/senha); Private Endpoint; criptografia em repouso (TDE) e auditoria nativas; Microsoft Defender for SQL; **Row-Level Security** nativa como segunda camada de isolamento; Prisma, usado na fatia vertical, suporta SQL Server.
- Contras: migração do schema e das migrations da fatia vertical; dialetos diferentes (T-SQL); o tier General Purpose tem latência de I/O maior que Business Critical.

## Decisão

Adotar a Opção C: **Azure SQL Database**, vCore **General Purpose**, provisionado, com **redundância de zona**, **acesso público desabilitado** (só Private Endpoint), **autenticação somente Microsoft Entra** e backup com redundância **ZRS**.

- **Identidade de banco:** cada Container App usa a própria Managed Identity como usuário de banco (`CREATE USER ... FROM EXTERNAL PROVIDER`). O Dashboard tem permissão só de leitura (`db_datareader`).
- **Isolamento de dados ([ADR-0001](0001-multi-tenancy-schema-por-tenant.md)):** o modelo schema-por-tenant continua possível no SQL Server (`CREATE SCHEMA`), mas o mecanismo muda. No PostgreSQL o roteamento era o `search_path`; no SQL Server não há equivalente por conexão. Por isso, se a multi-tenancy voltar, o isolamento passa a ser **Row-Level Security** com o tenant em `SESSION_CONTEXT`, aplicado pelo banco em toda consulta — mantendo a garantia que o ADR-0001 buscava: um bug de query não devolve dados de outro tenant. Com uma única empresa hoje, existe um único schema de aplicação.
- **Redundância de backup ZRS, não GRS:** a região par de Brazil South é South Central US (EUA). Com GRS, cópias de backup sairiam do Brasil — conflito com a LGPD e com o perfil de dados sensíveis de arquitetura corporativa. Aceita-se RPO/RTO maiores em desastre regional.

## Padrão arquitetural

**Database as a Service** com **identidade gerenciada no lugar de credenciais** (passwordless). O isolamento por tenant, quando necessário, passa do padrão *Bridge* (schema) para **Pool com Row-Level Security** — o banco continua sendo a camada que garante o isolamento, não a disciplina de cada query.

## Justificativa

O ArqFlow não usa nada exclusivo do PostgreSQL, então a troca custa só a migração do schema. Em troca, o banco passa a seguir a stack Microsoft da empresa, com autenticação só por Entra ID, Defender for SQL e auditoria nativos. Managed Instance resolveria um problema de compatibilidade que o ArqFlow não tem, a um custo maior.

## Consequências

- A fatia vertical (`arqflow-app`) troca o `provider` do Prisma de `postgresql` para `sqlserver` e regera as migrations; o ambiente local usa a imagem oficial do SQL Server no `docker-compose`. A conexão em Azure usa autenticação Entra (Managed Identity), sem senha na connection string.
- Revisar tipos específicos de PostgreSQL usados nas migrations (ex.: `uuid`, `text[]`, `jsonb`) e mapear para `uniqueidentifier`, tabelas filhas e `nvarchar(max)` com `ISJSON`.
- O isolamento por tenant do ADR-0001 deixa de ser "schema + `search_path`" e passa a ser Row-Level Security — atualizado nas consequências do ADR-0001 e do [ADR-0006](0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md).
- Custo: General Purpose 2 vCores com redundância de zona é a maior linha de compute da estimativa ([FinOps](../06-implantacao-e-operacao/estimativa-finops.md)); reserva de 1 ano e Azure Hybrid Benefit são as alavancas.

## Revisitar quando

Se a latência de I/O do General Purpose impedir o RNF de cotação < 1 s (improvável, já que a cotação lê do cache de preços, [ADR-0004](0004-motor-de-custeio-com-cache-de-precos.md)), avaliar Business Critical. Se o volume de leitura do Dashboard competir com a escrita, avaliar réplica de leitura.
