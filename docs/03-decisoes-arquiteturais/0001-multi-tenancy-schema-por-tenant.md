# ADR-0001: Multi-tenancy com schema separado por tenant, não linha compartilhada

**Status:** aceito
**Data:** 2026-02-03
**Decisores:** Arquitetura, Engenharia

## Contexto

O ArqFlow é vendido a múltiplas empresas-cliente (tenants), cada uma com dados de arquitetura corporativa potencialmente sensíveis (demandas, cenários, custos, diagramas). O discovery confirmou que o modelo é multi-tenant compartilhado (sem instância dedicada por cliente, ver [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#cada-cliente-vai-ter-sua-própria-instância-ou-é-tudo-compartilhado)), mas o time de Segurança/IAM dos clientes trata isolamento de dados como restrição inegociável, não preferência.

## Alternativas consideradas

### Opção A — Banco compartilhado com coluna `tenant_id` em cada tabela
- Prós: mais simples de operar (um único schema, migrações mais fáceis); custo de infraestrutura menor.
- Contras: isolamento depende inteiramente da aplicação lembrar de filtrar por `tenant_id` em toda consulta — um único bug de query (um `WHERE` esquecido) vaza dados entre clientes; mais difícil de provar para auditorias de segurança dos clientes.

### Opção B — Um schema PostgreSQL por tenant, mesmo banco físico
- Prós: isolamento reforçado pelo próprio banco de dados (uma conexão de aplicação usa um `search_path` fixo por tenant); um bug de query no pior caso falha (erro), não vaza dado silenciosamente entre schemas; mais fácil de demonstrar isolamento em auditoria de segurança do cliente.
- Contras: migrações de schema precisam rodar em N schemas (não é mais "uma migração"); número de schemas cresce com o número de clientes, exigindo automação de provisionamento desde o início.

## Decisão

Cada tenant recebe seu próprio schema PostgreSQL dentro da mesma instância de banco. O roteamento do schema correto acontece na borda (gateway de aplicação), a partir da identidade do tenant resolvida no login SSO.

## Justificativa

Como o Time de Segurança/IAM do cliente tem poder de veto sobre isolamento de dados (ver [stakeholders.md](../01-discovery/stakeholders.md)), a Opção A exigiria provar, para cada auditoria de cliente, que a aplicação nunca erra um filtro — uma garantia frágil por natureza. A Opção B desloca parte dessa garantia para o banco de dados, que é uma camada mais fácil de auditar e testar automaticamente (ex: testes que tentam acessar um schema errado e esperam falha).

## Consequências

- Toda migração de schema precisa ser aplicada de forma idempotente em todos os schemas de tenant — exige uma ferramenta de migração que itere sobre tenants, não apenas um `migrate` único.
- O onboarding self-service (RNF de até 10 minutos) precisa incluir a criação automatizada de um novo schema com a migração mais recente já aplicada — não é um passo manual.
- Consultas administrativas que cruzam todos os tenants (ex: métricas internas de uso do produto) ficam mais custosas, pois exigem iterar sobre schemas em vez de uma única consulta com `GROUP BY tenant_id`.

## Revisitar quando

Se o número de tenants crescer para a casa das milhares (o que tornaria a criação de um schema por tenant operacionalmente pesada no PostgreSQL), reavaliar migração para um modelo híbrido — schema dedicado apenas para os clientes maiores/mais sensíveis, e um modelo de linha compartilhada com políticas de *row-level security* nativas do PostgreSQL para os demais.
