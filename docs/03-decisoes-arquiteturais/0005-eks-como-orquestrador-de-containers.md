# ADR-0005: Orquestrar os serviços em Amazon EKS, não ECS Fargate

**Status:** aceito
**Data:** 2026-03-02
**Decisores:** Arquitetura

## Contexto

Os serviços internos do ArqFlow (ver [`04-diagramas/containers.png`](../04-diagramas/containers.png)) já nascem decompostos por domínio — Gateway de Identidade, Demanda/Cenário, Motor de Custeio, Serviço de Integração EA, Serviço de Dashboard — cada um com um perfil de carga diferente (o worker de integração com a BiZZdesign é assíncrono e em rajada, o motor de custeio precisa de resposta < 1s, o dashboard é majoritariamente leitura). Além disso, o ArqFlow é vendido a clientes enterprise cujos times de plataforma frequentemente já padronizaram em Kubernetes — em conversas de pré-venda e nas integrações com o time de EA do cliente, falar a mesma linguagem de plataforma facilita a avaliação técnica do produto. É preciso escolher onde e como esses serviços rodam em produção.

## Alternativas consideradas

### Opção A — Amazon ECS com Fargate
- Prós: menor sobrecarga operacional (sem control plane para se preocupar, integração nativa mais direta com IAM/ALB/CloudWatch); curva de aprendizado menor para o time; custo previsível por task.
- Contras: portabilidade limitada — um cliente enterprise com política de multi-cloud ou com exigência contratual de rodar componentes on-prem/em outra nuvem não tem caminho de migração razoável; ecossistema mais restrito para entrega progressiva (canary/blue-green) e para políticas de autoscaling orientadas a métricas de negócio (ex: profundidade de fila); menos alinhado ao vocabulário e às ferramentas que os times de plataforma dos clientes-alvo já usam.

### Opção B — Amazon EKS (control plane gerenciado pela AWS)
- Prós: portabilidade real (os mesmos manifests/Helm charts rodam, com ajustes mínimos, em outro cluster Kubernetes — relevante para um produto que vende governança de arquitetura a empresas que valorizam essa opcionalidade); ecossistema maduro para os requisitos não funcionais já assumidos (Argo CD para GitOps, Argo Rollouts para canary, Karpenter para bin-packing/custo, External Secrets Operator); permite modelar isolamento por domínio via namespaces + `NetworkPolicy`, estendendo à camada de compute o mesmo raciocínio de isolamento em camadas já usado no banco de dados (ADR-0001).
- Contras: mais complexidade operacional — mesmo com o control plane gerenciado, o time precisa de competência real em Kubernetes (RBAC, upgrades de versão, node groups, CNI); mais peças móveis do que ECS para o mesmo resultado.

## Decisão

Adotar Amazon EKS como orquestrador de containers para os serviços do ArqFlow, com node groups gerenciados para cargas estáveis e Karpenter para autoscaling de nós e consolidação por custo em cargas variáveis (ex: worker de integração EA).

## Padrão arquitetural

**Separação Control Plane / Data Plane** — o control plane do Kubernetes (gerenciado pela AWS) versus os node groups (data plane, sob nossa gestão) é a aplicação, na camada de orquestração, do mesmo princípio de responsabilidade dividida que já aparece no gateway de identidade (ADR-0002): delegar a uma camada especializada o que não é diferenciação do produto. O isolamento por domínio via namespace é uma instância de **Bulkhead Pattern** — detalhado no ADR-0006, que trata especificamente de como o multi-tenancy se traduz (ou não) para a camada de compute.

## Justificativa

A Opção A resolveria o problema imediato com menos esforço, mas o discovery do produto (ver [`stakeholders.md`](../01-discovery/stakeholders.md)) já identificou que o comprador do ArqFlow é o próprio time de arquitetura/EA do cliente — um público que avalia maturidade de plataforma como parte da decisão de compra. Optar por EKS é uma decisão tanto técnica (ecossistema de entrega progressiva e autoscaling orientado a métricas de negócio) quanto de posicionamento comercial (falar a língua de Kubernetes que esses times já falam).

## Consequências

- O time precisa investir em competência de Kubernetes (RBAC, upgrade de versão do cluster, gestão de node groups) — não é uma competência que se ganha de graça só por o control plane ser gerenciado.
- Custo fixo do control plane do EKS, independentemente de utilização, mais o custo dos node groups — exige que o dimensionamento (ADR relacionado a Karpenter/HPA, ver [`kubernetes-boas-praticas.md`](../06-implantacao-e-operacao/kubernetes-boas-praticas.md)) seja levado a sério para não pagar por capacidade ociosa.
- Toda decisão subsequente de deploy, isolamento multi-tenant em compute e entrega contínua (ADRs 0006 e 0007) passa a ser modelada em cima de primitivas Kubernetes, não das primitivas mais simples do ECS.

## Revisitar quando

Se a equipe permanecer pequena por um período prolongado e a promessa de portabilidade nunca for de fato exercida por nenhum cliente (ninguém pedir deploy fora da AWS ou citar isso como critério de decisão), reavaliar se a complexidade operacional do EKS se paga — nesse cenário, migrar os serviços mais simples e sem necessidade de canary/bin-packing fino de volta para ECS Fargate seria uma opção híbrida razoável.
