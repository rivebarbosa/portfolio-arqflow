# ADR-0012: Rodar os serviços em Azure Container Apps, não em AKS

**Status:** aceito
**Data:** 2026-11-02
**Decisores:** Arquitetura (Roberto Rivelino Barbosa)
**Substitui:** a escolha de compute do [ADR-0011](0011-migrar-plataforma-de-aws-eks-para-azure-aks.md) (AKS). A migração para Azure e a tabela de equivalência dos demais serviços continuam valendo.

## Contexto

O [ADR-0011](0011-migrar-plataforma-de-aws-eks-para-azure-aks.md) levou o ArqFlow para Azure e trocou EKS por AKS, carregando o argumento original do [ADR-0005](0005-eks-como-orquestrador-de-containers.md): Kubernetes como linguagem comum com times de plataforma de clientes enterprise e portabilidade entre nuvens.

Esse argumento era comercial e pertencia ao ArqFlow SaaS multi-tenant. Depois do pivô, o ArqFlow é uma aplicação de uma única empresa (ou um módulo de outra aplicação), dentro da arquitetura de referência corporativa em Azure ([ADR-0010](0010-bff-como-token-handler-com-entra-id-e-sensedia.md)). Ninguém mais compra o ArqFlow avaliando maturidade de Kubernetes, e não há exigência de rodar fora do Azure.

Também ficou decidido que a implantação e a operação devem ficar **na stack Microsoft**, usando serviços gerenciados sempre que possível. São cinco serviços de domínio pequenos (BFF, Demanda/Cenário, Motor de Custeio, Integração EA, Dashboard), mais um job diário de preços, mantidos por um time pequeno.

## Alternativas consideradas

### Opção A — Manter AKS (ADR-0011)
- Prós: nenhum retrabalho; ecossistema completo (Argo CD, Argo Rollouts, Kyverno, `NetworkPolicy`); controle total do runtime.
- Contras: o time opera um cluster (upgrades de versão, node pools, add-ons, CNI) para cinco serviços; custo fixo de nós ligados 24 h; a justificativa comercial do Kubernetes deixou de existir.

### Opção B — Azure App Service (Web App for Containers)
- Prós: o PaaS mais simples da Microsoft; deploy slots para troca sem indisponibilidade.
- Contras: sem escala orientada a eventos (o worker de Integração EA precisa escalar pelo tamanho da fila do Service Bus); não tem jobs agendados nativos para a atualização de preços; cada app ocupa capacidade do plano mesmo ocioso.

### Opção C — Azure Container Apps (ACA), em ambiente de workload profiles com VNet própria
- Prós: plataforma de containers serverless da Microsoft, sem cluster para operar; **KEDA embutido** (o worker escala pela fila do Service Bus e pode ficar em zero); **Container Apps Jobs** substituem o `CronJob` de preços; **revisões com divisão de tráfego** permitem blue/green e canary sem Argo Rollouts; Managed Identity por app; imagens puxadas do **Azure Container Registry** por identidade; ambiente interno com Private Link a partir do **Azure Front Door Premium**; agente OpenTelemetry gerenciado.
- Contras: menos controle que Kubernetes — não há `NetworkPolicy` entre apps do mesmo ambiente, nem admission controllers próprios (Kyverno), nem Argo CD; o ambiente com Private Endpoint cobra a taxa de gerenciamento do plano Dedicado; recursos máximos por réplica são menores que um nó de AKS.

## Decisão

Adotar a Opção C. Cada serviço de domínio vira um **Container App**; a atualização da tabela de preços vira um **Container Apps Job** agendado.

| Antes (AKS, ADR-0011) | Depois (Container Apps) |
| --- | --- |
| Cluster AKS + node pools + Node Auto Provisioning | Ambiente do Container Apps (workload profiles, perfil Consumption), VNet própria, redundância de zona |
| Namespace por domínio + `ResourceQuota`/`LimitRange` | Um Container App por domínio, com CPU/memória por réplica e mín./máx. de réplicas |
| `CronJob` de preços | Container Apps Job com gatilho agendado |
| KEDA (add-on do AKS) | Regras de escala do Container Apps (KEDA embutido), escala por fila do Service Bus |
| Argo Rollouts (canary) | Revisões múltiplas com divisão de tráfego por peso |
| Argo CD (GitOps) | Deploy de revisão pela esteira com OIDC — ver [ADR-0014](0014-entrega-continua-por-revisoes-do-container-apps.md) |
| Private Link Service sobre load balancer interno | Ambiente interno, acesso público desabilitado, Private Link do Front Door Premium para o ambiente |
| Entra Workload ID por `ServiceAccount` | Managed Identity atribuída por usuário, uma por Container App |
| External Secrets Operator | Referências a segredos do Key Vault no próprio Container App |
| Kyverno | Azure Policy (definições para Container Apps e ACR) + gates na esteira |
| Container Insights, Managed Prometheus, Managed Grafana | Log Analytics + Application Insights (agente OpenTelemetry gerenciado) + Azure Monitor |
| `NetworkPolicy` default-deny (Cilium) | Ingress interno ou desabilitado por app, criptografia peer-to-peer no ambiente, NSG na subnet e revalidação do JWT em cada serviço |

## Padrão arquitetural

**Serverless containers / Platform as a Service**: delegar à plataforma o que não diferencia o produto (orquestração, escala, rede de serviço). O **Bulkhead** do [ADR-0006](0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md) continua, agora na fronteira do Container App: cada domínio escala, falha e é publicado de forma independente.

## Justificativa

Para cinco serviços e um time pequeno, o custo de operar Kubernetes não é pago por nenhum requisito atual. O Container Apps entrega as três capacidades que de fato eram usadas no AKS — escala por eventos (KEDA), entrega progressiva (revisões) e identidade de workload (Managed Identity) — sem cluster, e se encaixa na arquitetura de referência (Front Door Premium com Private Link, Key Vault, Entra ID, ACR). Como a aplicação já é empacotada em containers, a escolha não prende o código: as mesmas imagens rodam em AKS se um gatilho futuro exigir.

## Consequências

- Etapa 06 reescrita: [`container-apps-boas-praticas.md`](../06-implantacao-e-operacao/container-apps-boas-praticas.md) substitui `kubernetes-boas-praticas.md`.
- O isolamento de rede entre serviços deixa de ser `NetworkPolicy` e passa a combinar ingress interno/desabilitado, NSG e zero trust na aplicação (todo serviço revalida o JWT, como já exigia o ADR-0010). Isso é mais fraco que default-deny em L3/L4 — está registrado como trade-off.
- GitOps com Argo CD deixa de ser aplicável. O modelo de entrega passa a ser decidido no [ADR-0014](0014-entrega-continua-por-revisoes-do-container-apps.md).
- O Private Link do Front Door exige ambiente de **workload profiles**, interno e com acesso público desabilitado; o Private Endpoint gera a taxa de gerenciamento do plano Dedicado, mesmo usando só o perfil Consumption (ver [FinOps](../06-implantacao-e-operacao/estimativa-finops.md)).
- O BFF mantém no mínimo 2 réplicas (sem escala a zero), para não adicionar cold start ao login.

## Revisitar quando

Se o ArqFlow precisar de isolamento de rede L3/L4 entre serviços, de controladores próprios (service mesh completo, operadores), de mais de ~15 serviços, ou de recursos por réplica acima dos limites do Container Apps — nesse caso, voltar a AKS com as mesmas imagens, reaproveitando o desenho do ADR-0011.
