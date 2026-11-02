# Estimativa FinOps da Infraestrutura

Ironia bem-vinda: o ArqFlow vende a capacidade de estimar custo de infraestrutura por cenário ([RF-03](../02-requisitos/requisitos-funcionais.md#rf-03--cotação-de-custo-finops-por-cenário), [ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md)). Este documento aplica a mesma disciplina à própria infraestrutura: uma estimativa mensal da arquitetura descrita em [`arquitetura-azure.md`](arquitetura-azure.md) — Container Apps, Azure SQL Database, Front Door Premium + WAF e Container Registry — com premissas e confiabilidade declaradas.

## Metodologia e premissas

- **Região:** Brazil South, preço sob demanda (pay-as-you-go), em US$, sem impostos, sem transferência de dados entre zonas.
- **Data de referência:** novembro de 2026.
- **Confiabilidade por linha:** a coluna "Fonte" diz de onde veio cada valor.
    - **Página** — publicado na página de preços ou na documentação do serviço.
    - **Aprox.** — ordem de grandeza a partir do preço público de referência (East US); Brazil South costuma ser mais cara. Precisa ser conferido na [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) antes de qualquer decisão de orçamento.
- **Estágio:** piloto em uma única empresa, tráfego baixo, uso em horário comercial, um ambiente de produção (dev/hml fora da conta).
- **Não incluído:** Sensedia e Entra ID (plataformas corporativas já licenciadas e compartilhadas entre aplicações), GitHub, pessoas, licença da BiZZdesign.

### Premissas de dimensionamento do Container Apps

- Réplicas mínimas sempre ligadas: BFF 2, Demanda/Cenário 2, Motor de Custeio 2, Dashboard 1 — 7 réplicas de 0,5 vCPU / 1 GiB (3,5 vCPU e 7 GiB no total).
- 30% do mês em uso ativo (horário comercial), 70% em tarifa *idle* (réplica parada no mínimo, sem processar requisições).
- Worker de Integração EA em zero na maior parte do tempo; job de preços com ~15 min/dia.
- Preços do plano Consumption (East US): vCPU ativo US$ 0,000024/s; memória US$ 0,000003/GiB·s; vCPU *idle* US$ 0,000003/s; memória *idle* US$ 0,000003/GiB·s. Franquia mensal de 180.000 vCPU·s, 360.000 GiB·s e 2 milhões de requisições por subscription.

## Baseline sob demanda

| Componente | Configuração | US$/mês | Fonte |
| --- | --- | --- | --- |
| Azure Container Apps — uso | Perfil Consumption, premissas acima (≈ US$ 141 de réplicas + ≈ 4 de worker/jobs − ≈ 5 de franquia) | ≈ 140 | Aprox. |
| Azure Container Apps — gerenciamento | Taxa do plano Dedicado (US$ 0,10/h), cobrada porque o ambiente tem Private Endpoint (Front Door), mesmo usando só Consumption | ≈ 73 | Página |
| Azure SQL Database | General Purpose, 2 vCores provisionados, redundância de zona, 32 GB, licença incluída, backup ZRS | ≈ 450 | Aprox. |
| Microsoft Defender for SQL | 1 servidor lógico | ≈ 15 | Aprox. |
| Azure Managed Redis | 2 instâncias pequenas (sessão e preços, separadas) | ≈ 80 | Aprox. (SKU a confirmar) |
| Azure Service Bus | Premium, 1 messaging unit (exigido para Private Endpoint) | ≈ 677 | Aprox. |
| Azure Front Door Premium + WAF | Taxa base (exigida para origem por Private Link) + requisições/dados de piloto; WAF incluído no Premium | ≈ 340 | Página (base US$ 330) |
| Private Endpoints | 8 (Azure SQL, 2× Redis, Service Bus, Key Vault, Blob, ACR, ambiente do Container Apps) | ≈ 58 | Aprox. |
| Azure Container Registry | Premium (exigido para Private Endpoint) | ≈ 50 | Aprox. |
| Microsoft Defender for Containers | Avaliação de vulnerabilidade das imagens no ACR | ≈ 10 | Aprox. (modelo de cobrança a confirmar) |
| NAT Gateway | 1 + ~50 GB processados | ≈ 35 | Aprox. |
| Log Analytics + Application Insights | ~15 GB ingeridos/mês | ≈ 35 | Aprox. |
| Key Vault + Blob Storage | operações e armazenamento de piloto | ≈ 4 | Aprox. |
| **Total estimado** | | **≈ US$ 1.970/mês** | |

**O que mais pesa:** Service Bus Premium (≈ 34%), Azure SQL (≈ 23%) e Front Door Premium (≈ 17%). Service Bus e Front Door estão na conta pelo mesmo motivo: a arquitetura de referência exige origens privadas, e nesses serviços Private Link/Private Endpoint só existe no tier Premium. É o custo explícito do princípio "nenhuma origem exposta à internet". O compute, que era a maior linha no AKS, virou uma das menores.

## Cenários otimizados

| Alavanca | O que muda | Economia | Trade-off |
| --- | --- | --- | --- |
| Reserva de 1 ano no Azure SQL | Compute do banco em capacidade reservada (≈ 35% de desconto, a confirmar por SKU) | ≈ US$ 140 | Compromisso de 1 ano — só depois que o piloto validar a continuidade |
| Azure Hybrid Benefit no Azure SQL | Usa licenças SQL Server com Software Assurance que a empresa já tenha | ≈ US$ 100 adicionais (a confirmar) | Só se houver licenças disponíveis; consome direitos de licença de outro uso |
| Azure savings plan for compute | Compromisso de gasto por hora em compute, aplicável ao Container Apps (elegibilidade e desconto a confirmar) | ≈ US$ 30 | Compromisso de 1 ou 3 anos; ganho pequeno porque o compute já é barato |
| Service Bus Standard | Troca Premium por Standard (≈ US$ 10/mês) | ≈ US$ 667 | Perde o Private Endpoint: o namespace fica acessível pelo endpoint público, protegido só por Entra ID (Managed Identity) e TLS. **Fere o princípio de origens privadas da referência — exige ADR de exceção** |
| Front Door compartilhado | Um perfil Front Door Premium para várias aplicações da empresa, com rotas por domínio | até US$ 330 por aplicação adicional | Governança do perfil compartilhado (dono, WAF comum) passa para o time de plataforma |

| | Baseline | Otimizado (mantém a referência) | Otimizado com exceção no Service Bus |
| --- | --- | --- | --- |
| **Total/mês** | ≈ US$ 1.970 | ≈ US$ 1.800 | ≈ US$ 1.130 |
| **Economia** | — | ≈ 9% | ≈ 43% |

O cenário do meio considera só a reserva do Azure SQL e o savings plan; o Azure Hybrid Benefit fica fora porque depende de licenças que talvez não existam. A recomendação para o piloto é esse cenário. A exceção no Service Bus deve ser decidida explicitamente, porque troca US$ 667/mês por uma origem pública.

## Custo da arquitetura de referência por aplicação

Com um único cliente (a própria empresa), a pergunta de FinOps que substitui o antigo "custo marginal por tenant" é: **quanto custa cada nova aplicação que adota a mesma arquitetura de referência?** Parte da conta já é compartilhada — Sensedia, Entra ID e, se a empresa decidir, o perfil do Front Door Premium. Com Container Apps, outra parte pode ser compartilhada: várias aplicações podem dividir o mesmo ambiente (e a mesma taxa de gerenciamento de US$ 73), cada uma com seus apps e identidades. O custo que fica com cada aplicação é o dos seus Container Apps, do Redis de sessão, do seu banco e dos Private Endpoints dos seus dados.

## Comparação com as estimativas anteriores

| Desenho | Total/mês | Observação |
| --- | --- | --- |
| AWS / EKS (`us-east-1`) | ≈ US$ 880 | Região mais barata, sem tiers Premium para origens privadas |
| Azure / AKS + PostgreSQL (out/2026) | ≈ US$ 2.160 | Nós de AKS ligados 24 h (≈ US$ 450 de compute) e Managed Prometheus/Grafana |
| **Azure / Container Apps + Azure SQL (este)** | **≈ US$ 1.970** | Compute ≈ US$ 213; banco mais caro que o PostgreSQL |

A troca de AKS por Container Apps reduz o compute em cerca de US$ 235/mês e elimina Managed Prometheus/Grafana e o load balancer interno; parte dessa economia é consumida pelo Azure SQL, mais caro que o PostgreSQL Flexible Server em configuração equivalente. Os números da AWS não são diretamente comparáveis: a região mudou e a arquitetura de referência exige tiers Premium para origens privadas.

## Confiabilidade desta estimativa

É uma estimativa de ordem de grandeza, suficiente para decidir arquitetura e discutir as alavancas acima, não uma fatura. Os preços do Container Apps vêm da referência East US; Azure SQL, Managed Redis e Defender precisam ser validados na Azure Pricing Calculator para Brazil South antes de virar orçamento. O consumo do Container Apps é sensível às premissas de horas ativas e de réplicas mínimas — por isso as premissas estão explícitas acima.

### Fontes consultadas (novembro de 2026)

- [Billing in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/billing) — modelo ativo/*idle*, franquia mensal e taxa de gerenciamento com Private Endpoint
- [Azure Container Apps pricing](https://azure.microsoft.com/pricing/details/container-apps/) — tabela de preços por região
- [Private Link do Front Door para Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/front-door-custom-virtual-network-private-link) — exige ambiente de workload profiles e Front Door Premium
- [Azure Front Door pricing](https://azure.microsoft.com/pricing/details/frontdoor/) — taxa base dos tiers Standard e Premium
- [Azure SQL Database pricing](https://azure.microsoft.com/pricing/details/azure-sql-database/single/) — vCore General Purpose
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) — referência para validar as linhas aproximadas
