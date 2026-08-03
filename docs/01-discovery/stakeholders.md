# Mapa de Stakeholders

Como o ArqFlow é um produto B2B, existem dois níveis de stakeholder: os **nossos** (internos, da empresa que constrói o ArqFlow) e os **do cliente** (dentro de cada empresa que contrata o produto). Misturar os dois níveis no discovery foi proposital — uma decisão de arquitetura como "como fazemos SSO" depende diretamente do que o time de segurança *do cliente* exige, não só do que o nosso time acha ideal.

| Stakeholder | Nível | O que precisa do sistema | Influência na arquitetura |
|---|---|---|---|
| **Arquiteto de soluções** (usuário-alvo) | Cliente | Fluxo único para demanda → cenário → custo → diagrama, sem trocar de ferramenta | Alto — define o modelo de dados central e a UX do fluxo crítico |
| **Time de FinOps do cliente** | Cliente | Visibilidade da estimativa de custo por cenário, antes da aprovação | Alto — define requisitos do motor de custeio |
| **Time de Segurança/IAM do cliente** | Cliente | Login via o provedor de identidade corporativo já existente (SSO), sem exceção | Alto — restrição de entrada, não negociável |
| **Time de Arquitetura Corporativa (EA) do cliente** | Cliente | Repositório de EA (BiZZdesign) atualizado automaticamente após decisões | Alto — define o contrato e a estratégia de integração |
| **Gestor/PMO do cliente** | Cliente | Dashboard de andamento das demandas por área/prioridade | Médio — define requisitos de relatórios e granularidade de dados |
| **Time comercial (nosso)** | Interno | Onboarding rápido de um novo cliente-tenant, sem trabalho manual de engenharia | Médio — influencia a decisão de multi-tenancy (self-service de provisionamento) |
| **Time de Engenharia (nosso)** | Interno | Conseguir evoluir o produto para múltiplos clientes sem reescrever integrações por cliente | Alto — motiva abstrair a integração de SSO e de EA por trás de interfaces genéricas |

## Por que este mapa importa para a arquitetura

Dois pontos mudaram decisões técnicas diretamente:

1. **O Time de Segurança/IAM do cliente tem poder de veto sobre o método de login.** Isso descarta, de saída, qualquer solução de autenticação própria com usuário/senha como único método — motivou tratar SSO federado como restrição de entrada, não feature opcional (ver [ADR-0002](../03-decisoes-arquiteturais/0002-federacao-de-identidade-via-gateway-oidc.md)).

2. **Cada cliente pode estar em um plano de API diferente da BiZZdesign** (alguns com acesso a webhooks, outros só a polling). Isso influenciou diretamente a escolha de um modelo de sincronização assíncrono e tolerante a atraso, em vez de uma integração síncrona que assumiria sempre o melhor caso (ver [ADR-0003](../03-decisoes-arquiteturais/0003-sincronizacao-assincrona-com-bizzdesign.md)).
