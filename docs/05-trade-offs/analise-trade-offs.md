# Análise de Trade-offs

## Isolamento forte vs. simplicidade operacional (multi-tenancy)

**O que escolhemos:** schema por tenant (ADR-0001) em vez de linha compartilhada com `tenant_id`.

**O que ganhamos:** uma garantia de isolamento que não depende de disciplina perfeita da aplicação em cada consulta — relevante em um produto que vende justamente confiança sobre dados sensíveis de arquitetura corporativa.

**O que perdemos:** simplicidade de migração de schema (agora multiplicada pelo número de tenants) e de consultas administrativas cross-tenant. Esse é um trade-off consciente de "mais trabalho operacional agora" em troca de "menos risco de vazamento entre clientes depois" — o tipo de troca que vale a pena quando o próprio modelo de negócio (B2B, vendendo a times de segurança/arquitetura) valoriza essa garantia mais do que a conveniência operacional.

## Token no navegador vs. sessão no servidor (identidade e BFF)

**O que escolhemos:** BFF como token handler, com Entra ID direto e Sensedia como API Gateway ([ADR-0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md)), substituindo o broker de identidade do ADR-0002.

**O que ganhamos:** nenhum token acessível por JavaScript — um XSS na SPA não leva access token nem refresh token. O cliente OAuth é confidencial e se autentica por certificado, sem client secret. A validação do JWT acontece em duas camadas independentes (Sensedia e serviço), e o broker, que deixou de ter função com um único IdP, sai do desenho.

**O que perdemos:** o BFF vira caminho crítico de todas as chamadas da SPA e passa a ter estado (sessão no Redis); o cookie de sessão reabre a necessidade de proteção CSRF; e existe um salto a mais (BFF) em toda chamada. É o mesmo princípio que aparece no resto do portfólio: preferir uma garantia estrutural (o token simplesmente não está no navegador) a uma garantia de processo (confiar que a SPA nunca terá um XSS).

*Decisão anterior, mantida como histórico:* o ADR-0002 delegava a federação de N IdPs a um broker — trade-off que fazia sentido enquanto o ArqFlow era SaaS multi-tenant com IdPs variados por cliente.

## Consistência forte vs. resiliência a um sistema externo (integração BiZZdesign)

**O que escolhemos:** sincronização assíncrona com retry (ADR-0003) em vez de chamada síncrona no momento da aprovação.

**O que ganhamos:** o fluxo crítico do produto (aprovar um cenário) nunca fica refém da disponibilidade de um sistema que não controlamos.

**O que perdemos:** simplicidade — "aprovado" e "sincronizado com a BiZZdesign" deixam de ser o mesmo evento, exigindo modelar e comunicar dois estados distintos ao usuário. Esse trade-off só é aceitável porque o discovery confirmou explicitamente que o time de EA do cliente já revisa manualmente essas sincronizações — se a expectativa fosse de sincronização instantânea e automática sem revisão humana, esse desenho precisaria ser reaberto.

## Precisão em tempo real vs. previsibilidade e velocidade (motor de custeio)

**O que escolhemos:** tabela de preços cacheada com defasagem de até 24h (ADR-0004) em vez de consulta síncrona às APIs de precificação de nuvem.

**O que ganhamos:** cotações instantâneas (RNF < 1s) e comparações de cenário deterministicamente reprodutíveis — dois cenários compostos no mesmo dia têm preços consistentes entre si.

**O que perdemos:** exatidão em tempo real, que nem sequer é o objetivo real deste componente (é uma ferramenta de decisão, não de faturamento). Esse é o trade-off mais claro dos quatro: a "precisão perfeita" seria estritamente pior para o caso de uso real, porque prejudicaria a comparação entre cenários ao introduzir variação de preço não relacionada a mudanças no próprio cenário.

## Controle de plataforma vs. simplicidade operacional (compute)

**O que escolhemos:** Azure Container Apps ([ADR-0012](../03-decisoes-arquiteturais/0012-azure-container-apps-como-plataforma-de-compute.md)), depois de dois desenhos com Kubernetes gerenciado — Amazon EKS ([ADR-0005](../03-decisoes-arquiteturais/0005-eks-como-orquestrador-de-containers.md)) e Azure AKS ([ADR-0011](../03-decisoes-arquiteturais/0011-migrar-plataforma-de-aws-eks-para-azure-aks.md)).

**O que ganhamos:** nenhum cluster para operar (sem upgrades de versão, node pools ou add-ons); escala por eventos (KEDA embutido), entrega progressiva (revisões com divisão de tráfego) e identidade de workload (Managed Identity) como recursos da plataforma; compute cerca de US$ 235/mês mais barato que o AKS ([FinOps](../06-implantacao-e-operacao/estimativa-finops.md)).

**O que perdemos:** controle fino. Não há `NetworkPolicy` entre apps do mesmo ambiente — o isolamento passa a combinar ingress interno/desabilitado, NSG e revalidação do JWT em cada serviço, que é mais fraco que default-deny em L3/L4. Também saem os controladores próprios (Argo CD, Argo Rollouts, Kyverno), substituídos por recursos da plataforma e por Azure Policy.

A justificativa do Kubernetes no ADR-0005 era comercial — falar a língua dos times de plataforma de clientes enterprise. Com o pivô para uma única empresa, essa força desapareceu, e o trade-off se inverteu. A portabilidade continua existindo na camada que importa: as mesmas imagens de container rodam em AKS se um gatilho do ADR-0012 aparecer.

## Ecossistema corporativo vs. custo e retrabalho (banco de dados)

**O que escolhemos:** Azure SQL Database ([ADR-0013](../03-decisoes-arquiteturais/0013-azure-sql-database-como-banco-relacional.md)) em vez de PostgreSQL Flexible Server.

**O que ganhamos:** banco alinhado à stack Microsoft da empresa — autenticação somente por Entra ID (sem senha), Defender for SQL, auditoria e Row-Level Security nativos.

**O que perdemos:** a fatia vertical (`arqflow-app`) precisa migrar o schema para T-SQL, e o banco fica mais caro que o PostgreSQL em configuração equivalente. O isolamento por tenant do [ADR-0001](../03-decisoes-arquiteturais/0001-multi-tenancy-schema-por-tenant.md), se voltar, troca `search_path` por Row-Level Security.

## Segurança estrutural vs. simplicidade (entrega contínua)

**O que escolhemos:** deploy por revisões do Container Apps, a partir do GitHub Actions com OIDC e aprovação por GitHub Environment ([ADR-0014](../03-decisoes-arquiteturais/0014-entrega-continua-por-revisoes-do-container-apps.md)), substituindo o GitOps com Argo CD ([ADR-0007](../03-decisoes-arquiteturais/0007-gitops-com-argocd-para-entrega-continua.md)).

**O que ganhamos:** nenhum segredo de longa duração com escrita em produção — o token do Entra ID é emitido por execução, expira em minutos e só sai para um job aprovado. Rollback em segundos (mover tráfego para a revisão anterior), sem um controlador a mais para operar.

**O que perdemos:** a reconciliação contínua do Argo CD. Drift não é corrigido sozinho; é impedido por Azure Policy e por *deployment stacks* com `denySettings`, e detectado por um `what-if` diário. Também deixa de ser verdade que "nenhuma credencial com escrita em produção existe fora do ambiente": ela existe, por minutos, dentro de um job aprovado. É a troca consciente que o próprio ADR-0007 deixou prevista para quando o controlador deixasse de se justificar.

## O padrão por trás desses trade-offs

Em três das quatro decisões (identidade, integração BiZZdesign, custeio), a escolha foi **não assumir que um sistema externo é tão confiável ou controlável quanto o próprio produto** — seja um IdP de cliente, a API da BiZZdesign, ou a API de precificação de uma nuvem. Essa é a lição central deste caso de estudo: arquitetura de um produto B2B que integra com terceiros passa mais tempo desenhando para a *falha e a variabilidade* desses terceiros do que desenhando o "caminho feliz" interno. Nas decisões de implantação ([ADR-0012](../03-decisoes-arquiteturais/0012-azure-container-apps-como-plataforma-de-compute.md)–[ADR-0014](../03-decisoes-arquiteturais/0014-entrega-continua-por-revisoes-do-container-apps.md)), o mesmo princípio reaparece em outra forma: preferir garantias estruturais (credencial que não é armazenada em lugar nenhum, banco sem senha, isolamento que não depende de disciplina de query, token que não existe no navegador — [ADR-0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md)) a garantias de processo — mesmo quando isso custa mais investimento operacional agora.
