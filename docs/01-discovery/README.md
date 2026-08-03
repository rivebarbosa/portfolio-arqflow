# Discovery

Assim como no primeiro caso de estudo deste portfólio, o objetivo aqui é responder três perguntas antes de desenhar qualquer diagrama: **qual problema de negócio estamos resolvendo, para quem, e o que "sucesso" significa em números.**

A diferença central deste caso: ArqFlow não é um sistema com um único dono (como um marketplace interno), é um **produto vendido a terceiros**. Isso muda o discovery — não basta entender o fluxo de trabalho do arquiteto, é preciso entender as restrições que cada cliente-empresa vai impor (o IdP dele, a ferramenta de EA que ele já usa, os dados que ele considera sensíveis).

1. [`contexto-negocio.md`](contexto-negocio.md) — o problema, objetivo de negócio e métricas de sucesso
2. [`stakeholders.md`](stakeholders.md) — mapa de stakeholders (incluindo os stakeholders *dos clientes*, não só os internos)
3. [`perguntas-discovery.md`](perguntas-discovery.md) — perguntas feitas, respostas obtidas e premissas assumidas

## Metodologia usada

- **Entrevistas com arquitetos de soluções** (personas-alvo do produto) sobre como hoje eles recebem demanda, montam cenários e cotam custo — normalmente em planilhas soltas e apresentações, sem rastreabilidade.
- **Levantamento de restrições de integração**: como funciona a API pública da BiZZdesign, quais protocolos de SSO os clientes-alvo já usam, quais APIs de precificação as clouds (AWS/Azure/GCP) expõem.
- **Priorização por "o que é restrição de terceiro, não escolha nossa"**: em um produto B2B que integra com sistemas de clientes, boa parte do discovery é mapear o que *não* podemos controlar (o IdP do cliente, o plano de API da BiZZdesign) antes de desenhar a arquitetura em torno disso.

Critério para sair do discovery: ter clareza sobre o fluxo de trabalho do arquiteto ponta a ponta, as restrições de integração de terceiros (mesmo que só em nível de "existe rate limit, não sabemos o número exato"), e o modelo de negócio (multi-tenant B2B, não single-tenant).
