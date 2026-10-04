# ADR-0011: Migrar a plataforma de AWS (EKS) para Azure (AKS)

**Status:** aceito
**Data:** 2026-10-03
**Decisores:** Arquitetura (Roberto Rivelino Barbosa)
**Substitui:** [ADR-0005](0005-eks-como-orquestrador-de-containers.md)

## Contexto

O ADR-0005 escolheu Amazon EKS quando o ArqFlow era um SaaS B2B vendido a vários clientes, e a escolha de nuvem era livre. Com o pivô para uma aplicação de uma única empresa, a nuvem passa a ser a da empresa. O [ADR-0010](0010-bff-como-token-handler-com-entra-id-e-sensedia.md) colocou o ArqFlow dentro da arquitetura de referência corporativa, que é desenhada sobre Azure: Front Door + WAF na borda, Blob Storage para a SPA, Entra ID como IdP, Key Vault para segredos, Azure Cache for Redis para sessão e Sensedia como API Gateway.

Manter os serviços em AWS enquanto identidade, borda e segredos ficam em Azure criaria um desenho multi-cloud sem motivo de negócio: tráfego entre nuvens, duas redes privadas para interligar, duas formas de identidade de workload e dois stacks de observabilidade.

## Alternativas consideradas

### Opção A — Manter EKS e tudo o mais em AWS, traduzindo a arquitetura de referência para equivalentes AWS
- Prós: nenhum retrabalho em ADRs 0005–0007 e na etapa 06.
- Contras: deixa o ArqFlow fora da arquitetura de referência (Front Door, Key Vault, Managed Identity); a integração com o Entra ID como IdP corporativo continua, mas sem a identidade de workload nativa (Entra Workload ID) nem Private Link para os serviços Azure da referência.

### Opção B — Híbrido: borda e identidade em Azure, serviços de domínio em EKS
- Prós: preserva o investimento em EKS.
- Contras: custo e latência de tráfego entre nuvens; duas redes privadas e dois modelos de IAM para operar; o pior dos dois mundos para um time pequeno.

### Opção C — Migrar toda a plataforma para Azure, com AKS no lugar do EKS
- Prós: aderência total à arquitetura de referência; uma rede (VNet com Private Endpoints), um modelo de identidade (Entra ID para pessoas e Workload ID para pods), um stack de observabilidade (Azure Monitor); o raciocínio do ADR-0005 a favor de Kubernetes (portabilidade, Argo CD, Argo Rollouts, KEDA) continua valendo no AKS.
- Contras: retrabalho de documentação (etapa 06, diagramas, FinOps); o time precisa conhecer as particularidades do AKS (Azure CNI, Workload ID, Node Auto Provisioning).

## Decisão

Adotar a Opção C. Equivalência dos componentes:

| Antes (AWS) | Depois (Azure) |
| --- | --- |
| Amazon EKS + node groups gerenciados | Azure Kubernetes Service (AKS) + node pools |
| Karpenter | Node Auto Provisioning do AKS (baseado em Karpenter) ou cluster autoscaler |
| CloudFront + WAF + S3 | Azure Front Door Premium + WAF + Blob Storage (Private Link) |
| ALB | Ingress interno no AKS, exposto ao Front Door por Private Link |
| — (não existia) | Sensedia API Gateway entre BFF e serviços (ADR-0010) |
| RDS PostgreSQL Multi-AZ | Azure Database for PostgreSQL Flexible Server com HA zone-redundant |
| ElastiCache Redis | Azure Cache for Redis (sessão do BFF e cache de preços, instâncias separadas) |
| SNS + SQS + DLQ | Azure Service Bus (tópicos, assinaturas e dead-letter queue) |
| IRSA | Microsoft Entra Workload ID |
| Secrets Manager + KMS | Azure Key Vault (+ External Secrets Operator ou Secrets Store CSI Driver) |
| ECR | Azure Container Registry |
| CloudWatch, AMP, AMG | Azure Monitor (Container Insights, Log Analytics), Azure Monitor managed Prometheus, Azure Managed Grafana |
| Terraform state em S3 + DynamoDB | Terraform state em Azure Storage (lock por blob lease) |

## Padrão arquitetural

O mesmo do ADR-0005: **separação Control Plane / Data Plane** (control plane gerenciado pela Microsoft, node pools sob nossa gestão) e **Bulkhead** por namespace de domínio (ADR-0006). A mudança de nuvem não altera o padrão, só o provedor — que é justamente a portabilidade que o ADR-0005 usou como argumento a favor do Kubernetes.

## Justificativa

Com identidade, borda e segredos definidos em Azure pela arquitetura de referência, a única forma de evitar um desenho multi-cloud acidental é trazer também o compute. A escolha por Kubernetes feita no ADR-0005 é o que torna a migração barata: Helm charts, Argo CD, Argo Rollouts, KEDA e Kyverno são os mesmos no AKS.

## Consequências

- Etapa 06 reescrita para Azure: [`arquitetura-azure.md`](../06-implantacao-e-operacao/arquitetura-azure.md), [`kubernetes-boas-praticas.md`](../06-implantacao-e-operacao/kubernetes-boas-praticas.md), [`esteira-cicd.md`](../06-implantacao-e-operacao/esteira-cicd.md) e [`estimativa-finops.md`](../06-implantacao-e-operacao/estimativa-finops.md).
- ADR-0006 e ADR-0007 continuam válidos; só mudam as referências de EKS/ECR para AKS/ACR.
- O namespace `identidade` (broker) dá lugar ao namespace `bff`.
- A Sensedia é licenciada pela empresa e compartilhada entre aplicações; seu custo não entra na estimativa FinOps do ArqFlow.

## Revisitar quando

Se o ArqFlow voltar a ser vendido a clientes que exijam outra nuvem, reavaliar com base nos mesmos manifests Kubernetes — trocando os serviços gerenciados da tabela acima pelos equivalentes do provedor.
