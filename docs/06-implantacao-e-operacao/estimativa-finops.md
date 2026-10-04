# Estimativa FinOps da Infraestrutura

Ironia bem-vinda: o ArqFlow vende a capacidade de estimar custo de infraestrutura por cenário ([RF-03](../02-requisitos/requisitos-funcionais.md#rf-03--cotação-de-custo-finops-por-cenário), [ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md)). Este documento aplica a mesma disciplina à própria infraestrutura: uma estimativa mensal da arquitetura descrita em [`arquitetura-azure.md`](arquitetura-azure.md), com premissas e confiabilidade declaradas.

## Metodologia e premissas

- **Região:** Brazil South, preço sob demanda (pay-as-you-go), em US$, sem impostos, sem transferência de dados entre zonas.
- **Data de referência:** outubro de 2026.
- **Confiabilidade por linha:** a coluna "Fonte" diz de onde veio cada valor.
    - **API** — consultado na [Azure Retail Prices API](https://prices.azure.com/api/retail/prices) para Brazil South em 03/10/2026.
    - **Página** — publicado na página de preços do serviço.
    - **Aprox.** — ordem de grandeza a partir do preço público de referência (East US); Brazil South costuma ser mais cara. Precisa ser conferido na [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) antes de qualquer decisão de orçamento.
- **Estágio:** piloto em uma única empresa, tráfego baixo, um ambiente de produção (dev/hml fora da conta).
- **Não incluído:** Sensedia e Entra ID (plataformas corporativas já licenciadas e compartilhadas entre aplicações), pessoas, licença da BiZZdesign.

## Baseline sob demanda

| Componente | Configuração | US$/mês | Fonte |
| --- | --- | --- | --- |
| AKS — control plane | Tier Standard (SLA), 1 cluster | ≈ 73 | Aprox. |
| Node pool estável | 3× `Standard_D2s_v5` Linux (1 por zona), US$ 0,153/h | 335 | API |
| Node Auto Provisioning (Spot) | ~2× `Standard_D2s_v5` Spot, US$ 0,028274/h | 41 | API |
| PostgreSQL Flexible Server | General Purpose 2 vCores, HA zone-redundant, 128 GiB | ≈ 290 | Aprox. |
| Azure Cache for Redis | 2× Standard C0 (sessão e preços, instâncias separadas) | ≈ 80 | Aprox. |
| Azure Service Bus | Premium, 1 messaging unit (exigido para Private Endpoint) | ≈ 677 | Aprox. |
| Azure Front Door Premium | Taxa base (exigida para origem por Private Link) + requisições/dados de piloto | ≈ 340 | Página (base US$ 330) |
| Private Endpoints | 7 (PostgreSQL, 2× Redis, Service Bus, Key Vault, Blob, ACR) | ≈ 51 | Aprox. |
| Azure Container Registry | Premium (exigido para Private Endpoint) | ≈ 50 | Aprox. |
| NAT Gateway | 1 + ~50 GB processados | ≈ 35 | Aprox. |
| Load balancer interno + Private Link Service | Standard LB (o Private Link Service não tem cobrança própria) | ≈ 18 | Aprox. |
| Microsoft Defender for Containers | 6 vCores | ≈ 42 | Aprox. |
| Log Analytics | ~15 GB ingeridos/mês | ≈ 25 | Aprox. |
| Azure Monitor managed Prometheus | cluster pequeno | ≈ 15 | Aprox. |
| Azure Managed Grafana | Standard, ~5 usuários | ≈ 80 | Aprox. |
| Key Vault + Blob Storage | operações e armazenamento de piloto | ≈ 4 | Aprox. |
| **Total estimado** | | **≈ US$ 2.160/mês** | |

**O que mais pesa:** Service Bus Premium (≈ 31%) e Front Door Premium (≈ 16%). Os dois estão na conta pelo mesmo motivo: a arquitetura de referência exige origens privadas, e nesses serviços Private Link/Private Endpoint só existe no tier Premium. É o custo explícito do princípio "nenhuma origem exposta à internet".

## Cenários otimizados

| Alavanca | O que muda | Economia | Trade-off |
| --- | --- | --- | --- |
| Reserva de 1 ano | Node pool estável e compute do PostgreSQL passam para instância reservada (≈ 35% de desconto, a confirmar por SKU) | ≈ US$ 210 | Compromisso de 1 ano — só depois que o piloto validar a continuidade |
| Service Bus Standard | Troca Premium por Standard (≈ US$ 10/mês) | ≈ US$ 667 | Perde o Private Endpoint: o namespace fica acessível pelo endpoint público, protegido só por Entra ID (managed identity) e TLS. **Fere o princípio de origens privadas da referência — exige ADR de exceção** |
| Front Door compartilhado | Um perfil Front Door Premium para várias aplicações da empresa, com rotas por domínio | até US$ 330 por aplicação adicional | Governança do perfil compartilhado (dono, WAF comum) passa para o time de plataforma |

| | Baseline | Otimizado (mantém a referência) | Otimizado com exceção no Service Bus |
| --- | --- | --- | --- |
| **Total/mês** | ≈ US$ 2.160 | ≈ US$ 1.950 | ≈ US$ 1.280 |
| **Economia** | — | ≈ 10% | ≈ 41% |

A recomendação para o piloto é o cenário do meio: manter a referência e aplicar só a reserva. A exceção no Service Bus deve ser decidida explicitamente, porque troca US$ 667/mês por uma origem pública.

## Custo da arquitetura de referência por aplicação

Com um único cliente (a própria empresa), a pergunta de FinOps que substitui o antigo "custo marginal por tenant" é: **quanto custa cada nova aplicação que adota a mesma arquitetura de referência?** Parte da conta já é compartilhada — Sensedia, Entra ID e, se a empresa decidir, o perfil do Front Door Premium. O custo que fica com cada aplicação é o do BFF, do Redis de sessão e dos Private Endpoints dos seus próprios dados, não o da borda inteira.

## Comparação com a estimativa anterior (AWS)

A estimativa para AWS/EKS em `us-east-1` era ≈ US$ 880/mês. Os números não são comparáveis diretamente: a região mudou (Brazil South costuma ser mais cara que regiões dos EUA) e a arquitetura de referência exige tiers Premium para origens privadas que o desenho AWS não tinha. A diferença é, na maior parte, o preço do isolamento de rede e da borda gerenciada — não do compute.

## Confiabilidade desta estimativa

É uma estimativa de ordem de grandeza, suficiente para decidir arquitetura e discutir as alavancas acima, não uma fatura. Apenas as linhas marcadas como **API** e **Página** foram conferidas em fonte primária; as demais precisam ser validadas na Azure Pricing Calculator para Brazil South antes de virar orçamento.

### Fontes consultadas (outubro de 2026)

- [Azure Retail Prices API](https://prices.azure.com/api/retail/prices) — `Standard_D2s_v5` Linux e Spot, Brazil South
- [Azure Front Door pricing](https://azure.microsoft.com/pricing/details/frontdoor/) — taxa base dos tiers Standard e Premium
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) — referência para validar as linhas aproximadas
