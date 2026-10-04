# ADR-0010: BFF como token handler, Microsoft Entra ID direto e Sensedia como API Gateway

**Status:** aceito
**Data:** 2026-10-03
**Decisores:** Arquitetura (Roberto Rivelino Barbosa)
**Substitui:** [ADR-0002](0002-federacao-de-identidade-via-gateway-oidc.md)

## Contexto

O ADR-0002 foi escrito para um ArqFlow SaaS B2B multi-tenant, em que cada empresa-cliente trazia o próprio IdP (Azure AD, Okta, Google Workspace). Para isso, um broker OIDC/SAML federava todos os IdPs e entregava um token normalizado.

Duas coisas mudaram:

1. **Pivô de escopo.** O ArqFlow passou a ser uma aplicação de uma única empresa (podendo também virar um módulo de outra aplicação em microsserviços). Existe um único IdP corporativo, o **Microsoft Entra ID**. O broker deixa de resolver um problema real e vira só mais um ponto único de falha.
2. **Arquitetura de referência.** A empresa adotou uma arquitetura de referência para toda aplicação web com SPA: **SPA + BFF + OAuth 2.0/JWT, com Azure Front Door, Entra ID e Sensedia API Gateway**. O ArqFlow precisa segui-la.

Além disso, o desenho atual não tinha camada de borda para as APIs: a SPA chamaria os serviços de domínio diretamente, com o token no navegador. Isso expõe o access token a XSS e a extensões maliciosas — risco incompatível com uma ferramenta que guarda dados sensíveis de arquitetura corporativa.

## Alternativas consideradas

### Opção A — Manter o broker (Keycloak) e a SPA como cliente público, com o token no navegador
- Prós: nenhuma mudança em relação ao ADR-0002; SPA chama as APIs direto, sem um salto a mais.
- Contras: access token (e possivelmente refresh token) acessível por JavaScript — um XSS leva o token; o broker continua como componente crítico, agora sem a justificativa multi-IdP; foge da arquitetura de referência da empresa.

### Opção B — Token-mediating backend: um backend obtém os tokens e entrega o access token à SPA
- Prós: refresh token fica no servidor; cliente OAuth confidencial.
- Contras: o access token continua no navegador, então um XSS ainda consegue usá-lo enquanto ele for válido.

### Opção C — BFF como cliente OAuth confidencial, com sessão no servidor; Entra ID direto; Sensedia entre o BFF e os serviços
- Prós: nenhum token acessível pelo navegador (só um cookie `HttpOnly`); cliente confidencial autenticado por certificado ou Managed Identity, sem client secret; Sensedia centraliza validação de JWT, rate limit e quotas; segue a arquitetura de referência; remove o broker.
- Contras: o BFF vira caminho crítico de todas as chamadas da SPA; sessão com estado exige cache distribuído (Redis); o cookie reabre a necessidade de proteção CSRF.

## Decisão

Adotar a Opção C:

- A SPA chama apenas rotas `/bff/*` no mesmo domínio, via Azure Front Door. Nunca chama a Sensedia nem os serviços de domínio diretamente.
- O **BFF** é o cliente OAuth 2.0 confidencial no **Entra ID**: Authorization Code + PKCE, autenticação do cliente por certificado (`private_key_jwt`) ou Managed Identity com federated credential. Tokens ficam no **Azure Cache for Redis**, criptografados, com TTL. O navegador recebe só um cookie `__Host-` `HttpOnly; Secure; SameSite`.
- O BFF chama a **Sensedia** com `Authorization: Bearer <JWT>`. A Sensedia valida o JWT offline (JWKS em cache: assinatura, `iss`, `aud`, `exp`, scopes) e aplica rate limit e quotas.
- Cada serviço de domínio (Demanda/Cenário, Motor de Custeio, Dashboard) **revalida o JWT** e aplica a autorização fina (app roles do Entra ID: Solicitante, Arquiteto, Gestor/PMO).
- O broker de identidade do ADR-0002 é removido.

Os parâmetros obrigatórios (vida do token, cookie, CSRF, headers, logout) seguem a seção "Padrões de segurança obrigatórios" da arquitetura de referência e estão resumidos nos [requisitos não funcionais](../02-requisitos/requisitos-nao-funcionais.md#identidade-e-acesso).

## Padrão arquitetural

**Backend for Frontend (BFF)** no papel de **token handler**, conforme o draft do IETF *OAuth 2.0 for Browser-Based Applications*: o BFF é o único cliente OAuth, e o navegador só conhece uma sessão. Somado a **Zero Trust entre camadas** — Gateway e serviços validam o mesmo JWT de forma independente, sem confiar na camada anterior só por estar na rede interna. A Sensedia cumpre o papel de **API Gateway** (roteamento, políticas, rate limit), que o ADR-0002 deixava explicitamente fora do escopo do broker.

## Justificativa

Com um único IdP, o motivo do broker (federar N provedores) desaparece, e o que sobra é o risco de manter um componente de autenticação a mais. Ao mesmo tempo, o maior risco real do desenho anterior não estava na federação, e sim no token dentro do navegador. A Opção C é a única em que um XSS não leva token nenhum, e é exatamente o padrão que a empresa definiu para novas aplicações — o ArqFlow, sendo a ferramenta do próprio time de arquitetura, deveria ser o primeiro exemplo de conformidade.

## Consequências

- O BFF precisa de alta disponibilidade (no mínimo 2 réplicas, `PodDisruptionBudget`, health probes no Front Door).
- Novo componente de dados: Azure Cache for Redis para sessão, separado do cache de preços do Motor de Custeio (ADR-0004), para que um não consiga despejar as chaves do outro.
- Proteção CSRF obrigatória nas rotas `/bff/*` que alteram estado (SameSite + header customizado ou anti-forgery token).
- RF-07 muda de "IdP configurado por tenant" para "Entra ID corporativo". A autorização passa a usar app roles do Entra ID em vez do mapeamento de claims por tenant que o ADR-0002 previa.
- A fatia vertical do ADR-0008 (JWT + bcrypt com senha local) diverge desta decisão. Ela continua válida como passo de aprendizado, mas a evolução do `arqflow-app` deve trocar o login local pelo fluxo BFF + Entra ID.
- Multi-tenancy (ADR-0001 e ADR-0006) não é revisto aqui. Com o pivô para empresa única, esses ADRs precisam ser reavaliados em um ADR próprio.

## Revisitar quando

Se o ArqFlow voltar a atender mais de uma empresa com IdPs diferentes (por exemplo, como módulo vendido a terceiros), reavaliar se o Entra ID External ID ou um broker volta a ser necessário — mantendo o BFF como token handler em qualquer cenário.
