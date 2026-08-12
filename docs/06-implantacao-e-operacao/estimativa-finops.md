# Estimativa FinOps da Infraestrutura

Ironia bem-vinda: o ArqFlow vende aos seus clientes a capacidade de estimar custo de infraestrutura por cenário ([RF-03](../02-requisitos/requisitos-funcionais.md#rf-03--cotação-de-custo-finops-por-cenário), [ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md)). Este documento aplica a mesma disciplina à própria infraestrutura do produto: uma estimativa de custo mensal para a arquitetura descrita em [`arquitetura-aws.md`](arquitetura-aws.md), com premissas explícitas, exatamente como o RNF de custeio do produto exige das estimativas que ele mesmo gera.

## Metodologia e premissas

- **Região:** `us-east-1`, preço sob demanda (*on-demand*), sem desconto de suporte AWS, sem impostos, sem tarifa de transferência de dados entre AZs.
- **Data de referência dos preços:** agosto de 2026, levantados via pesquisa (ver fontes ao final). Assim como o Motor de Custeio do próprio ArqFlow, esta é uma estimativa com data de referência declarada, não uma cotação em tempo real — a mesma justificativa do [ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md) (defasagem aceitável em troca de velocidade e reprodutibilidade) se aplica aqui.
- **Estágio do produto:** piloto, 3–5 tenants (consistente com o contexto já assumido no [ADR-0002](../03-decisoes-arquiteturais/0002-federacao-de-identidade-via-gateway-oidc.md)), tráfego baixo, sem carga de produção em escala.
- **Não incluído:** custo de desenvolvimento/pessoas, licenciamento de terceiros (ex: eventual custo da própria integração com a BiZZdesign), ambientes de dev/staging (esta estimativa cobre um único ambiente de produção).

## Baseline sob demanda

| Componente | Configuração | Custo/mês (US$) |
| --- | --- | --- |
| EKS — control plane | 1 cluster | 73,00 |
| EC2 — node group gerenciado | 3× `m6g.large` (1 por AZ, carga estável) | 168,63 |
| EC2 — Karpenter (spot) | ~2× `m6g.large` equivalente, rajada (worker de integração EA, jobs) | 33,58 |
| RDS PostgreSQL Multi-AZ | `db.r6g.large` + 100GB gp3 (schema por tenant, ADR-0001) | 340,00 |
| ElastiCache Redis | `cache.t4g.medium`, nó único (cache de preços, ADR-0004) | 47,45 |
| NAT Gateway | 3× (1 por AZ) + ~50GB processados/mês | 100,80 |
| ALB | base + ~1 LCU médio | 22,27 |
| CloudFront + S3 (frontend) | dentro do free tier (< 1TB/mês) | 3,00 |
| SNS + SQS + DLQ | dentro do free tier (< 1M mensagens/mês) | 0,00 |
| Secrets Manager | ~15 segredos (credenciais de banco, tokens BiZZdesign/IdP por tenant) | 7,00 |
| ECR | ~20GB de imagens de container | 2,00 |
| Route 53 | 1 hosted zone + queries | 1,50 |
| KMS | 2–3 CMKs + requests | 3,00 |
| CloudWatch Logs | ~15GB ingeridos/mês (após free tier) | 5,50 |
| Amazon Managed Prometheus | cluster pequeno (~5 nós, cardinalidade moderada) | 40,00 |
| Amazon Managed Grafana | 2 editores + 3 leitores | 33,00 |
| **Total estimado** | | **≈ US$ 880/mês** |

## Cenário otimizado

Aplicando três alavancas de FinOps clássicas — sem mudar nenhuma decisão arquitetural, só o modelo de compra e uma escolha de disponibilidade:

| Alavanca | O que muda | Trade-off |
| --- | --- | --- |
| Savings Plan / Reserved Instance (1 ano, sem pagamento antecipado) | EC2 do node group gerenciado, RDS e ElastiCache passam de sob demanda para reservado | Compromisso de 1 ano — só vale a pena porque o piloto já validou que o produto continua em operação |
| NAT Gateway único (em vez de 1 por AZ) | Reduz de 3 para 1 NAT Gateway | Perda de redundância de egress por AZ — aceitável em piloto, **não** aceitável quando o RNF de disponibilidade de 99,5% valer para todos os tenants em produção geral |
| Nenhuma mudança nos serviços gerenciados de baixo custo (Secrets Manager, ECR, Route 53, KMS) | — | Já estão perto do piso de custo possível; otimizar aqui não move a agulha |

| | Baseline sob demanda | Cenário otimizado |
| --- | --- | --- |
| **Total/mês** | ≈ US$ 880 | ≈ US$ 654 |
| **Economia** | — | ≈ 26% (US$ 227/mês) |

## Custo marginal por tenant adicional

Esta é a linha que mais vale a pena levar para uma conversa de pré-vendas: como o compute é compartilhado por domínio e não replicado por tenant ([ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md)), o custo marginal de aceitar mais um cliente-piloto é, em essência, o custo de mais um schema e mais alguns segredos — não o custo de mais um conjunto inteiro de containers.

- Armazenamento adicional (novo schema): ~2–5GB × US$ 0,115/GB ≈ **US$ 0,25–0,60/mês**
- Segredos adicionais (client secret do IdP, token da BiZZdesign se aplicável): 2 × US$ 0,40 ≈ **US$ 0,80/mês**
- Compute: absorvido pela capacidade já provisionada até um limiar de escala (o HPA/Karpenter só adiciona nó quando a utilização real exigir)

**Custo marginal estimado por tenant, em regime de piloto: ~US$ 1–2/mês**, até o próximo limiar de escala de compute (cada nó adicional de `m6g.large` amortizado entre todos os tenants ativos naquele momento, ~US$ 56/mês dividido pelo número de tenants).

Essa é a validação numérica, em dólares, da alternativa que o [ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md) rejeitou: se o isolamento multi-tenant tivesse sido feito por namespace-por-tenant (replicando o conjunto completo de serviços a cada cliente novo), o custo marginal por tenant seria da ordem de **centenas de dólares por mês** (um EKS node group e réplicas de todos os 6 serviços por cliente), não de dólares — o mesmo argumento de escala que já havia rejeitado banco dedicado por tenant no ADR-0001, agora quantificado para a camada de compute.

## Confiabilidade desta estimativa

Assim como o ArqFlow exibe a data de referência de cada estimativa de custo ao usuário (RF-03), este documento declara a sua: os valores acima foram levantados via pesquisa em agosto de 2026 e são uma estimativa de ordem de grandeza — suficiente para uma decisão de arquitetura e para uma conversa de pré-vendas, não uma fatura. Para uma cotação vinculante, o próximo passo é montar esta mesma configuração na [AWS Pricing Calculator](https://calculator.aws) ou solicitar um *Well-Architected Review*/estimativa formal via AWS.

### Fontes consultadas (agosto de 2026)

- [Amazon EKS Pricing](https://aws.amazon.com/eks/pricing/) — control plane
- [EC2 On-Demand Instance Pricing](https://aws.amazon.com/ec2/pricing/on-demand/) — `m6g.large`
- [Amazon RDS Pricing](https://aws.amazon.com/rds/postgresql/pricing/) — `db.r6g.large` Multi-AZ, storage gp3
- [Amazon ElastiCache Pricing](https://aws.amazon.com/elasticache/pricing/) — `cache.t4g.medium`
- [Amazon VPC Pricing](https://aws.amazon.com/vpc/pricing/) — NAT Gateway
- [Elastic Load Balancing Pricing](https://aws.amazon.com/elasticloadbalancing/pricing/) — ALB / LCU
- [Amazon CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/) — transferência de dados
- [AWS Secrets Manager Pricing](https://aws.amazon.com/secrets-manager/pricing/)
- [Amazon ECR Pricing](https://aws.amazon.com/ecr/pricing/)
- [Amazon Route 53 Pricing](https://aws.amazon.com/route53/pricing/)
- [Amazon CloudWatch Pricing](https://aws.amazon.com/cloudwatch/pricing/) — Logs
- [Amazon Managed Service for Prometheus](https://aws.amazon.com/prometheus/pricing/) e [Amazon Managed Grafana Pricing](https://aws.amazon.com/grafana/pricing/)
- [Amazon SQS Pricing](https://aws.amazon.com/sqs/pricing/) e [Amazon SNS Pricing](https://aws.amazon.com/sns/pricing/)
