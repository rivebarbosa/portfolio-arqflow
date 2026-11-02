# Implantação e Operação

Esta seção documenta como o ArqFlow roda em produção, **inteiramente na stack Microsoft**: a infraestrutura Azure de referência, como os serviços são organizados e operados no Azure Container Apps, e como uma mudança sai do commit até o usuário. Continua os ADRs [0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md) (identidade), [0012](../03-decisoes-arquiteturais/0012-azure-container-apps-como-plataforma-de-compute.md) (compute), [0013](../03-decisoes-arquiteturais/0013-azure-sql-database-como-banco-relacional.md) (banco) e [0014](../03-decisoes-arquiteturais/0014-entrega-continua-por-revisoes-do-container-apps.md) (entrega contínua).

## Stack de implantação e operação

| Função | Serviço Microsoft |
| --- | --- |
| Borda, CDN e WAF | Azure Front Door Premium + WAF (regras gerenciadas) |
| Gerenciador de containers | Azure Container Apps (ambiente de workload profiles, interno) |
| Registro de imagens | Azure Container Registry (Premium, Private Endpoint) |
| Banco relacional | Azure SQL Database (General Purpose, redundância de zona) |
| Mensageria | Azure Service Bus (Premium) |
| Cache (sessão e preços) | Azure Managed Redis |
| Arquivos estáticos da SPA | Azure Blob Storage |
| Identidade | Microsoft Entra ID (pessoas e Managed Identities) |
| Segredos | Azure Key Vault |
| Observabilidade | Azure Monitor, Log Analytics, Application Insights |
| Segurança e governança | Microsoft Defender for Cloud, Azure Policy |
| CI/CD e IaC | GitHub Actions (OIDC com Entra ID) e Bicep |

A Sensedia é a única peça fora da stack Microsoft: é o API Gateway corporativo exigido pela arquitetura de referência, compartilhado entre aplicações.

| Documento | O que você vai encontrar |
| --- | --- |
| [`arquitetura-azure.md`](arquitetura-azure.md) | Arquitetura de referência no Azure — borda, identidade, rede, Container Apps, dados, segurança, observabilidade — com diagrama |
| [`container-apps-boas-praticas.md`](container-apps-boas-praticas.md) | Como os serviços são organizados e operados: apps e jobs, identidade, rede, segredos, escala, revisões |
| [`esteira-cicd.md`](esteira-cicd.md) | A esteira de CI/CD: da branch ao deploy em produção, com gates de qualidade e segurança |
| [`estimativa-finops.md`](estimativa-finops.md) | Estimativa de custo mensal (baseline e otimizado), com os trade-offs de cada alavanca |

## Como esta seção se conecta ao resto do portfólio

- A borda e a identidade seguem a arquitetura de referência SPA + BFF + OAuth 2.0/JWT ([ADR-0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md)): Front Door + WAF, BFF como token handler, Entra ID como IdP e Sensedia como API Gateway.
- A mensageria assíncrona do [ADR-0003](../03-decisoes-arquiteturais/0003-sincronizacao-assincrona-com-bizzdesign.md) (fila + retry para a BiZZdesign) se materializa em Azure Service Bus, com dead-letter queue e alerta no Azure Monitor.
- O motor de custeio ([ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md)) é um Container App próprio, e a atualização diária da tabela de preços é um Container Apps Job agendado, escrevendo num Redis separado do Redis de sessão do BFF.
- O Bulkhead por domínio do [ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md) passa de namespace para Container App.
