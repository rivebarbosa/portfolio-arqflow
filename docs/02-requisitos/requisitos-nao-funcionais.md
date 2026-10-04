# Requisitos Não Funcionais

## Isolamento multi-tenant

| Requisito | Alvo | Origem |
|---|---|---|
| Isolamento de dados entre tenants | Nenhuma consulta pode retornar dados de outro tenant, nem em caso de bug de aplicação — o isolamento precisa existir também na camada de dados, não só na de aplicação | [contexto-negocio.md](../01-discovery/contexto-negocio.md#restrições-conhecidas-desde-o-início) — motivou [ADR-0001](../03-decisoes-arquiteturais/0001-multi-tenancy-schema-por-tenant.md) |
| Onboarding de novo tenant | Self-service, sem intervenção de engenharia, em até 10 minutos | [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#cada-cliente-vai-ter-sua-própria-instância-ou-é-tudo-compartilhado) |

## Identidade e acesso

Seguem a arquitetura de referência SPA + BFF + OAuth 2.0/JWT ([ADR-0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md)).

| Requisito | Alvo | Origem |
|---|---|---|
| Provedor de identidade | Microsoft Entra ID corporativo, via OIDC (Authorization Code + PKCE) | Pivô para empresa única + arquitetura de referência |
| Tokens no navegador | Nenhum: access, refresh e ID token ficam no BFF; o navegador recebe só um cookie `__Host-` `HttpOnly; Secure; SameSite` | ADR-0010 |
| Autenticação do cliente OAuth | Certificado ou federated credential (Workload ID); client secret proibido em produção | ADR-0010 |
| Vida do access token | 5 a 15 minutos, renovado pelo BFF com rotação do refresh token, sem ação do usuário | Arquitetura de referência |
| Sessão | Expiração por inatividade de 30 min e absoluta de 8 h; ID de sessão renovado após o login | Arquitetura de referência |
| Validação do JWT | Na Sensedia e em cada serviço: assinatura via JWKS em cache, `iss`, `aud`, `exp`/`nbf`, scopes/app roles | ADR-0010 (zero trust entre camadas) |
| Proteção CSRF | Obrigatória em toda rota `/bff/*` que altera estado | Consequência do cookie de sessão |
| Autorização | App roles do Entra ID (Solicitante, Arquiteto, Gestor/PMO), aplicadas no serviço | RF-07 |

## Integração com sistemas de terceiros (BiZZdesign)

| Requisito | Alvo | Origem |
|---|---|---|
| Tolerância a indisponibilidade da API da BiZZdesign | O fluxo de aprovação do arquiteto nunca é bloqueado por indisponibilidade da BiZZdesign | [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#a-sincronização-com-a-bizzdesign-precisa-ser-em-tempo-real) |
| Prazo de sincronização | Até 24h entre aprovação e sincronização bem-sucedida, com re-tentativa automática | RF-06 |
| Visibilidade de falha de sincronização | Toda falha de sincronização é visível ao arquiteto/EA do cliente, nunca silenciosa | Requisito derivado — sincronização assíncrona não pode significar "sem garantia nenhuma" |

## Custeio (FinOps)

| Requisito | Alvo | Origem |
|---|---|---|
| Atualização da tabela de preços de nuvem | No máximo 24h de defasagem em relação ao preço público das clouds | [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#a-cotação-de-custo-finops-precisa-refletir-o-preço-da-nuvem-naquele-segundo-exato) |
| Latência de cálculo de estimativa de custo | < 1s ao adicionar/remover um recurso do cenário (cálculo local sobre tabela cacheada, não chamada externa síncrona) | Consequência direta de [ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md) |

## Disponibilidade e desempenho

| Requisito | Alvo | Origem |
|---|---|---|
| Disponibilidade do fluxo de registro/cotação de demanda | 99,5% — é uma ferramenta de produtividade interna dos clientes, não um sistema transacional de missão crítica | Natureza do produto (B2B, uso em horário comercial) |
| Atualização do dashboard de andamento | Tolerância de até 5 minutos de atraso em relação ao estado real | RF-08 |

## Segurança

| Requisito | Alvo | Origem |
|---|---|---|
| Dados sensíveis de arquitetura corporativa | Criptografados em repouso; nunca compartilhados entre tenants mesmo em backups/restauração | [stakeholders.md](../01-discovery/stakeholders.md) (Time de Segurança/IAM do cliente) |
| Auditoria de ações | Toda aprovação de cenário e sincronização com terceiros é registrada com usuário, timestamp e resultado | RF-06, natureza de ferramenta de governança |

---

## Nota sobre premissas não confirmadas

O RNF de atualização da API da BiZZdesign assume estabilidade razoável do contrato de API — documentado como premissa em [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#premissas-assumidas-perguntas-sem-resposta-definitiva-no-discovery), não como fato confirmado por SLA formal.
