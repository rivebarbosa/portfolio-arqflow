# Portfólio de Arquitetura de Software — ArqFlow

Este repositório documenta, do início ao fim, o processo de arquitetura de um produto fictício: **ArqFlow**, uma plataforma SaaS B2B para times de arquitetura de soluções gerenciarem o ciclo de vida de uma demanda técnica — do recebimento da demanda até a aprovação de um cenário de solução com custo estimado (FinOps) e desenho de arquitetura.

Este é o segundo caso de estudo do meu portfólio. O primeiro ([PagFácil](https://github.com/rivebarbosa/portfolio-arquitetura)) trata de um domínio transacional/financeiro; este trata de um domínio diferente — uma ferramenta B2B multi-tenant de governança e produtividade para arquitetos — para mostrar como o mesmo processo de arquitetura se adapta a restrições muito diferentes (aqui, multi-tenancy, federação de identidade corporativa e integração com ferramentas de terceiros são as forças dominantes, não consistência transacional).

> 💡 Projeto fictício, criado para fins de portfólio.

## Sobre mim

Sou **Roberto Rivelino Barbosa**, Arquiteto de Soluções Sênior, com experiência na definição de padrões arquiteturais (microsserviços, event-driven, SOA/ESB, API-first) e na tradução de necessidades de negócio em soluções alinhadas à estratégia de TI, em ambientes AWS e Azure. Atuo na governança de arquitetura sob o framework TOGAF, com vivência em Enterprise Architecture (Ardoq, Qualiware, ArchiMate) e foco constante em segurança e compliance (LGPD, PCI-DSS, Zero Trust).

- LinkedIn: [linkedin.com/in/rivelinoroberto](https://linkedin.com/in/rivelinoroberto)
- E-mail: roberto.rivebarbosa@gmail.com

## Como navegar

| Etapa | Documento | O que você vai encontrar |
|---|---|---|
| 1. Discovery | [`docs/01-discovery/`](docs/01-discovery/README.md) | Contexto de negócio, stakeholders, perguntas feitas e premissas assumidas |
| 2. Requisitos | [`docs/02-requisitos/`](docs/02-requisitos/requisitos-funcionais.md) | Requisitos funcionais e não funcionais, com critérios de aceite mensuráveis |
| 3. Decisões | [`docs/03-decisoes-arquiteturais/`](docs/03-decisoes-arquiteturais/) | ADRs — cada decisão relevante, alternativas consideradas e motivo da escolha |
| 4. Diagramas | [`docs/04-diagramas/`](docs/04-diagramas/README.md) | Modelo C4 (contexto e containers), feitos em draw.io |
| 5. Trade-offs | [`docs/05-trade-offs/analise-trade-offs.md`](docs/05-trade-offs/analise-trade-offs.md) | O que foi ganho e perdido em cada decisão relevante |

## Resumo do produto

ArqFlow é vendido a empresas (clientes B2B, cada uma um "tenant") cujos times de arquitetura de soluções usam a ferramenta para:

1. **Receber uma demanda** — uma área de negócio ou produto abre uma solicitação de solução técnica.
2. **Criar cenários possíveis** — o arquiteto modela 2+ abordagens técnicas alternativas para atender a demanda.
3. **Cotar cada cenário**, incluindo análise de **FinOps** — estimativa de custo de infraestrutura/licenciamento por cenário, comparável entre nuvens.
4. **Anexar desenhos de arquitetura** a cada cenário (diagramas C4/técnicos).
5. **Integrar com ferramentas de EA já usadas pelo cliente**, como **BiZZdesign**, para manter o repositório de arquitetura corporativa sincronizado.
6. **Autenticar via SSO** — cada empresa-cliente conecta seu próprio provedor de identidade (Azure AD, Okta, etc.).
7. **Acompanhar tudo em dashboards** de andamento (quantas demandas, em que fase, SLA de resposta, custo aprovado vs. estimado).

**Principais desafios de arquitetura abordados:**
- Isolamento de dados entre clientes (multi-tenancy) em um produto que lida com informação sensível de arquitetura corporativa
- Federação de identidade com provedores de SSO variados por cliente, sem acoplar o core do produto a um único protocolo
- Integração com um sistema de terceiros (BiZZdesign) que a ArqFlow não controla, com garantias de consistência realistas
- Estimativa de custo (FinOps) que precisa ser rápida e comparável entre cenários, sem depender de chamada síncrona às APIs de precificação das clouds a cada cotação

## Stack proposta



- Backend: serviços organizados por domínio (linguagem a definir por você)
- Backend: serviços organizados por domínio, com API REST e contrato versionado
- Integração assíncrona: fila de eventos para sincronização com o BiZZdesign e para o motor de custos, com idempotência no consumidor
- Dados: PostgreSQL com schema por tenant, mais cache das tabelas de preço de nuvem
- Identidade: broker OIDC/SAML para federação com o provedor de cada cliente

---

*Portfólio criado para demonstrar processo de arquitetura de software. Não representa um sistema em produção nem uma parceria real com a BiZZdesign.*
