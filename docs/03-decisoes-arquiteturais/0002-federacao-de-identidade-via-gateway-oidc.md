# ADR-0002: Federar identidade via gateway OIDC/SAML central, não integrar cada IdP diretamente

**Status:** aceito
**Data:** 2026-02-05
**Decisores:** Arquitetura, Engenharia

## Contexto

Os 3 clientes-piloto já usam três provedores de identidade diferentes (Azure AD, Okta, Google Workspace), e o discovery deixou claro que essa variedade só tende a crescer com novos clientes (ver [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#quantos-provedores-de-identidade-diferentes-precisamos-suportar-no-dia-1)). RF-07 exige que o login aconteça exclusivamente via o IdP do próprio tenant, sem senha própria no ArqFlow.

## Alternativas consideradas

### Opção A — Implementar a integração de cada provedor diretamente no backend do ArqFlow
- Prós: controle total sobre o fluxo, sem dependência de componente externo.
- Contras: cada novo provedor de identidade (ou cada nova nuance de configuração de um cliente) exige mudança de código no core do produto; acopla a lógica de negócio do ArqFlow a detalhes de protocolo de autenticação de terceiros.

### Opção B — Um gateway de identidade dedicado (compatível com OIDC/SAML) na frente do produto, que federa qualquer IdP do cliente e entrega ao ArqFlow um token único e normalizado
- Prós: o core do produto lida com um único formato de identidade normalizada, independente de qual IdP o cliente usa; adicionar suporte a um novo provedor é configuração no gateway, não mudança de código de negócio; ferramentas desse tipo (ex: Keycloak, Auth0) já implementam as nuances de cada protocolo.
- Contras: introduz um componente de infraestrutura crítico adicional (se o gateway cair, ninguém loga); exige modelar cuidadosamente o mapeamento de "identidade federada" para "tenant + papel" dentro do ArqFlow.

## Decisão

Adotar um gateway de identidade dedicado como camada de federação: cada tenant configura seu IdP (OIDC ou SAML) no gateway via painel administrativo self-service; o ArqFlow recebe sempre um token normalizado (OIDC) do gateway, nunca fala diretamente com o IdP do cliente.

## Justificativa

A resposta do discovery — múltiplos provedores hoje, mais no futuro, sem lista fechada — é exatamente o cenário em que integração ponto-a-ponto no core do produto não escala: cada cliente novo viraria uma tarefa de engenharia. Um gateway de federação transforma "suportar um novo IdP" em configuração, alinhado ao requisito de onboarding self-service.

## Consequências

- O gateway de identidade se torna um ponto único de falha para autenticação — exige alta disponibilidade dedicada e monitoramento próprio, com RNF de disponibilidade potencialmente mais rígido que o resto do produto.
- A modelagem de "papel do usuário dentro do tenant" (arquiteto, gestor, admin) precisa ser resolvida a partir de claims do token federado (grupos do IdP, por exemplo) — exige acordo de mapeamento por tenant durante o onboarding, não é automático para 100% dos casos.
- Existe uma dependência de operação de um componente de terceiros (o software do gateway), com curva de aprendizado e necessidade de manter atualizado por causa de vulnerabilidades de segurança em componentes de autenticação.

## Revisitar quando

Se surgir a necessidade de suportar um método de login que não seja OIDC/SAML (ex: um cliente que exige autenticação por certificado mTLS), reavaliar se o gateway escolhido suporta essa extensão ou se é necessário um componente adicional específico.
