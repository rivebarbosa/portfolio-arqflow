# ADR-0008: Stack de implementação e escopo da primeira fatia vertical (código real)

**Status:** aceito
**Data:** 2026-09-21
**Decisores:** Arquitetura (Roberto Rivelino Barbosa)

> O código descrito por este ADR vive em um repositório GitHub separado,
> **`arqflow-app`** — de propósito, não neste repositório de
> documentação. Ver a justificativa dessa separação no `README.md` da
> raiz do repositório `arqflow-app`.

## Contexto

Os ADRs 0001–0007 descrevem a arquitetura-alvo do ArqFlow em escala de produto SaaS B2B real: multi-tenancy, federação de identidade corporativa, EKS, GitOps com Argo CD, integração com BiZZdesign. Essa é a visão de arquitetura de referência do portfólio, mas ela não é o ponto de partida executável — é o destino.

Chegou o momento de sair do protótipo estático (HTML/CSS/JS, sem backend, usado até aqui para validar fluxo e UX) e começar a construir código real, rodável, que efetivamente implemente uma fatia do produto. Isso traz um contexto diferente dos ADRs anteriores: quem vai escrever e manter este código no dia a dia é o próprio autor do portfólio, que se declara não familiarizado com o ecossistema Node.js. Construir logo de cara a arquitetura-alvo completa (multi-tenant, Kubernetes, broker de identidade) tornaria o código ilegível para quem está aprendendo a stack, e adiaria indefinidamente a existência de qualquer coisa funcional.

Portanto, a pergunta deste ADR não é "qual a arquitetura final do ArqFlow" (já respondida nos ADRs anteriores), e sim: **qual stack e qual escopo tornam viável uma primeira fatia vertical real, mantendo o caminho aberto para a arquitetura-alvo, sem forçar complexidade que ainda não se paga?**

## Alternativas consideradas

### Opção A — Implementar já a arquitetura-alvo completa (multi-tenant, EKS, Argo CD, broker OIDC)
- Prós: zero retrabalho arquitetural mais à frente; coerência total e imediata com os ADRs 0001–0007.
- Contras: complexidade muito acima do necessário para uma primeira fatia; exige conhecimento avançado de Kubernetes/GitOps que não é o foco desta etapa; inviabiliza o requisito explícito de "código simples o suficiente para eu (Roberto) compreender"; tempo até ter algo executável aumenta drasticamente.

### Opção B — Stack mínima "de brinquedo" (sem TypeScript, sem ORM, SQL cru, sem containerização)
- Prós: poucas peças móveis, curva de aprendizado baixíssima.
- Contras: abre mão de garantias importantes (tipagem, migrações versionadas) que na prática TORNAM o código mais fácil de entender e de dar manutenção, não mais difícil; não atende ao requisito explícito de containerização e dos três ambientes; distancia-se demais da arquitetura-alvo, exigindo reescrita e não evolução.

### Opção C — Fatia vertical completa (Auth + Projetos/Demandas), com stack "enterprise, mas simples": NestJS + DDD em camadas leve + Prisma + PostgreSQL + React + Docker, deixando multi-tenancy, EKS, GitOps e integrações externas para iterações futuras
- Prós: entrega algo real, ponta a ponta (login, CRUD de demandas, containerizado, com banco relacional) já nesta etapa; TypeScript + Prisma dão tipagem e migrações versionadas — reduzem erros exatamente para quem está aprendendo; DDD em camadas (domain/application/infrastructure/presentation) sem padrões pesados (sem CQRS, sem event sourcing) preserva a intenção arquitetural dos ADRs anteriores em escala reduzida; o caminho para a arquitetura-alvo fica aberto (ver seção "Consequências").
- Contras: existe uma distância consciente entre o que este ADR implementa e o que os ADRs 0001–0007 descrevem — precisa ficar explícito para não confundir quem ler o repositório depois.

## Decisão

Adotar a Opção C. A primeira fatia vertical real do ArqFlow é implementada com:

- **Backend:** NestJS (TypeScript), organizado em Domain-Driven Design por módulo/contexto delimitado (`modules/auth`, `modules/projetos`), com quatro camadas por módulo — `domain` (entidades e interfaces de repositório, sem dependência de framework), `application` (casos de uso e DTOs), `infrastructure` (implementações concretas com Prisma, segurança), `presentation` (controllers, guards, decorators HTTP).
- **Persistência:** PostgreSQL, acessado via Prisma ORM (escolhido por gerar um cliente totalmente tipado e por ter um fluxo de migração simples — `prisma migrate dev` — mais acessível a quem não conhece o ecossistema do que escrever SQL/migrations à mão ou configurar o TypeORM).
- **Autenticação:** JWT + bcrypt, com um fluxo de "primeiro acesso" (cria o admin inicial), login, troca de senha e recuperação de senha por e-mail — este último usando uma porta (`EmailService`) com um adapter de console em desenvolvimento, trocável por um provedor real futuramente sem tocar em regra de negócio.
- **Front-end:** React + Vite + TypeScript, com TanStack Query para estado de servidor (evita a complexidade de um Redux para este escopo) e React Router para navegação.
- **Containerização:** Docker multi-stage (`development`/`production`) para backend e front-end, orquestrados localmente via `docker-compose.yml`.
- **Escopo funcional desta fatia:** módulo de Identidade & Acesso (usuários, papéis, login) e módulo de Projetos/Demandas (CRUD, com o campo "Status da Análise" já validado no protótipo). Os demais módulos do ciclo de vida (Requisitos, Decisões/ADRs, Diagramas, Trade-offs, Implantação) ficam para iterações futuras, uma de cada vez.

Multi-tenancy, federação de identidade corporativa, orquestração em Kubernetes/EKS, GitOps com Argo CD e integração com BiZZdesign — todos definidos nos ADRs 0001, 0002, 0003, 0005, 0006 e 0007 — **não são implementados nesta fatia**. Continuam sendo a arquitetura-alvo documentada; este ADR não os revoga, apenas define o que é construído primeiro e por quê.

## Padrão arquitetural

**Arquitetura em camadas inspirada em DDD/Hexagonal, deliberadamente sem os padrões mais avançados** (sem CQRS, sem Result/Either monads, sem event sourcing). A camada de `domain` nunca importa nada do NestJS ou do Prisma — só a camada de `infrastructure` conhece esses detalhes, através de interfaces (`UserRepository`, `DemandaRepository`, `EmailService`) definidas em `domain`/`shared`. Isso preserva o valor central do DDD (isolar a regra de negócio de detalhes técnicos) sem impor vocabulário/complexidade que atrapalhariam a legibilidade para quem está aprendendo Node.js.

## Justificativa

O requisito mais restritivo declarado para esta etapa foi explícito: "não sou conhecedor do Node.js, faça o possível para ter um código simples que eu possa compreender". Ao mesmo tempo, o portfólio inteiro existe para demonstrar qualidade arquitetural — as duas coisas juntas apontam para a Opção C: usar boas práticas reais (tipagem, camadas, portas/adapters, migrações versionadas), mas escalonar a AMBIÇÃO da arquitetura, não a sua qualidade. Prisma, especificamente, foi escolhido sobre TypeORM/Knex por gerar tipos automaticamente a partir do `schema.prisma` — menos código boilerplate de mapeamento para o Roberto escrever e entender.

## Consequências

- Existe uma lacuna consciente entre este ADR e os ADRs 0001–0007 (nenhuma multi-tenancy, nenhuma federação SSO, nenhum EKS). Qualquer pessoa lendo o repositório precisa entender que a fatia atual é uma implementação inicial, não a arquitetura-alvo final — este ADR e o README de cada pasta cumprem esse papel.
- Ao evoluir para multi-tenant (ADR-0001), o schema Prisma do módulo de Projetos precisará ganhar uma coluna/estratégia de isolamento por tenant, e o módulo de Auth precisará acomodar múltiplas organizações — refatoração esperada, não um erro desta decisão.
- A ausência de Kubernetes/EKS nesta fase significa que "Homologação" e "Produção" (ver ADR-0009) ainda não têm um destino de infraestrutura real — o pipeline de CI/CD já builda e publica as imagens Docker, mas o passo de implantação de fato fica marcado como pendente até a infraestrutura de nuvem ser contratada/definida.
- Prisma prende o projeto ao seu próprio ecossistema de migração (não é "apenas SQL"); aceito conscientemente pelo ganho de tipagem e simplicidade para quem está aprendendo.

## Revisitar quando

Quando a segunda fatia vertical (ex.: módulo de Decisões/ADRs ou Diagramas) estiver sendo iniciada, ou quando o produto precisar atender mais de uma empresa-cliente de fato (gatilho para reabrir a discussão de multi-tenancy do ADR-0001 dentro deste código real).
