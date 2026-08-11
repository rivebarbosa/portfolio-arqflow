# Análise de Trade-offs

## Isolamento forte vs. simplicidade operacional (multi-tenancy)

**O que escolhemos:** schema por tenant (ADR-0001) em vez de linha compartilhada com `tenant_id`.

**O que ganhamos:** uma garantia de isolamento que não depende de disciplina perfeita da aplicação em cada consulta — relevante em um produto que vende justamente confiança sobre dados sensíveis de arquitetura corporativa.

**O que perdemos:** simplicidade de migração de schema (agora multiplicada pelo número de tenants) e de consultas administrativas cross-tenant. Esse é um trade-off consciente de "mais trabalho operacional agora" em troca de "menos risco de vazamento entre clientes depois" — o tipo de troca que vale a pena quando o próprio modelo de negócio (B2B, vendendo a times de segurança/arquitetura) valoriza essa garantia mais do que a conveniência operacional.

## Controle total vs. delegação a um componente especializado (identidade)

**O que escolhemos:** gateway de identidade federada (ADR-0002) em vez de integrar cada IdP diretamente no core do produto.

**O que ganhamos:** a complexidade de suportar N provedores de identidade não cresce linearmente com o código de negócio do ArqFlow — cresce na configuração de um componente especializado nisso.

**O que perdemos:** um ponto único de falha adicional (se o gateway cair, ninguém loga em nenhum tenant) e uma dependência de operar bem um componente de infraestrutura de segurança, que não é o core business do produto. Esse é o trade-off clássico de "build vs. delegar para um componente especializado" — aceito porque autenticação corporativa é um domínio maduro (protocolos padronizados), diferente de, por exemplo, o motor de custeio, que é uma diferenciação real do produto e por isso foi construído sob medida.

## Consistência forte vs. resiliência a um sistema externo (integração BiZZdesign)

**O que escolhemos:** sincronização assíncrona com retry (ADR-0003) em vez de chamada síncrona no momento da aprovação.

**O que ganhamos:** o fluxo crítico do produto (aprovar um cenário) nunca fica refém da disponibilidade de um sistema que não controlamos.

**O que perdemos:** simplicidade — "aprovado" e "sincronizado com a BiZZdesign" deixam de ser o mesmo evento, exigindo modelar e comunicar dois estados distintos ao usuário. Esse trade-off só é aceitável porque o discovery confirmou explicitamente que o time de EA do cliente já revisa manualmente essas sincronizações — se a expectativa fosse de sincronização instantânea e automática sem revisão humana, esse desenho precisaria ser reaberto.

## Precisão em tempo real vs. previsibilidade e velocidade (motor de custeio)

**O que escolhemos:** tabela de preços cacheada com defasagem de até 24h (ADR-0004) em vez de consulta síncrona às APIs de precificação de nuvem.

**O que ganhamos:** cotações instantâneas (RNF < 1s) e comparações de cenário deterministicamente reprodutíveis — dois cenários compostos no mesmo dia têm preços consistentes entre si.

**O que perdemos:** exatidão em tempo real, que nem sequer é o objetivo real deste componente (é uma ferramenta de decisão, não de faturamento). Esse é o trade-off mais claro dos quatro: a "precisão perfeita" seria estritamente pior para o caso de uso real, porque prejudicaria a comparação entre cenários ao introduzir variação de preço não relacionada a mudanças no próprio cenário.

## Portabilidade e maturidade de plataforma vs. simplicidade operacional (orquestração)

**O que escolhemos:** Amazon EKS (ADR-0005) em vez de ECS Fargate.

**O que ganhamos:** portabilidade real para um cluster Kubernetes fora da AWS se algum dia isso for exigido, e um ecossistema maduro (GitOps, entrega progressiva, autoscaling orientado a métricas de negócio) que fala a mesma língua dos times de plataforma dos clientes-alvo — relevante em um produto avaliado tecnicamente por esses mesmos times.

**O que perdemos:** simplicidade operacional. EKS exige competência real de Kubernetes do time — RBAC, upgrade de versão, gestão de node groups — que o ECS Fargate dispensaria quase inteiramente. Esse é um trade-off consciente de "mais investimento em capacitação de plataforma agora" em troca de uma opcionalidade e de um ecossistema que só se pagam se o produto de fato escalar em número de serviços e de clientes enterprise sensíveis a portabilidade.

## Segurança estrutural vs. simplicidade de um único pipeline (entrega contínua)

**O que escolhemos:** GitOps com Argo CD (ADR-0007) em vez da esteira de CI aplicar mudanças diretamente no cluster.

**O que ganhamos:** nenhuma credencial de escrita em produção precisa existir fora do cluster — a esteira nunca tem esse poder, o que elimina uma classe inteira de risco de vazamento de credencial. Como consequência colateral, também ganhamos correção automática de *drift* e rollback como uma operação de Git, não de re-execução de pipeline.

**O que perdemos:** um componente a mais para operar dentro do cluster, e uma etapa extra de latência entre "a imagem foi publicada" e "a mudança está de fato em produção" — que passa a depender do ciclo de reconciliação do Argo CD, não só da duração do pipeline. Coerente com o mesmo raciocínio de segurança que já orienta o resto do produto: preferir uma garantia estrutural (a credencial simplesmente não existe fora do cluster) a uma garantia de processo (disciplina de rotacionar/restringir uma credencial que existe).

## O padrão por trás desses trade-offs

Em três das quatro decisões (identidade, integração BiZZdesign, custeio), a escolha foi **não assumir que um sistema externo é tão confiável ou controlável quanto o próprio produto** — seja um IdP de cliente, a API da BiZZdesign, ou a API de precificação de uma nuvem. Essa é a lição central deste caso de estudo: arquitetura de um produto B2B que integra com terceiros passa mais tempo desenhando para a *falha e a variabilidade* desses terceiros do que desenhando o "caminho feliz" interno. Nas decisões de implantação (orquestração e entrega contínua, [ADR-0005](../03-decisoes-arquiteturais/0005-eks-como-orquestrador-de-containers.md)–[ADR-0007](../03-decisoes-arquiteturais/0007-gitops-com-argocd-para-entrega-continua.md)), o mesmo princípio reaparece em outra forma: preferir garantias estruturais (credencial que não existe fora do cluster, isolamento que não depende de disciplina de query) a garantias de processo — mesmo quando isso custa mais investimento operacional agora.
