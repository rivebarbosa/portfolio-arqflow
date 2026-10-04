# Arquitetura Azure de Referência

Esta arquitetura implanta os containers de [`04-diagramas/containers.png`](../04-diagramas/containers.png) sobre Azure, com AKS como orquestrador ([ADR-0011](../03-decisoes-arquiteturais/0011-migrar-plataforma-de-aws-eks-para-azure-aks.md)) e o padrão **SPA + BFF + OAuth 2.0/JWT** da arquitetura de referência corporativa ([ADR-0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md)). Três preocupações guiam o desenho: nenhum token no navegador, nenhuma origem exposta à internet além do Front Door, e resiliência a sistemas de terceiros indisponíveis (BiZZdesign, APIs de precificação).

## Diagrama

![Diagrama de Implantação (C4 - Deployment) - ArqFlow em Azure/AKS](../04-diagramas/implantacao-c4.svg)

*(Nível C4 de Implantação: mostra onde cada container roda. Como eles se comunicam está no [Diagrama de Containers](../04-diagramas/containers.png); o fluxo de login está no [diagrama de sequência](../04-diagramas/sequencia-autenticacao.svg).)*

**Leitura do diagrama:** a SPA é servida pelo Azure Front Door Premium a partir do Blob Storage, sob WAF com regras gerenciadas (OWASP). As rotas `/bff/*` vão do Front Door ao BFF por Private Link. O BFF chama a Sensedia com o JWT; a Sensedia encaminha aos serviços de domínio no AKS por rede privada e mTLS. Banco, caches, Service Bus, Key Vault e registry só são alcançáveis por Private Endpoint. A saída para a BiZZdesign e para as APIs de precificação passa por um NAT Gateway com IP fixo (útil para allowlist do lado dos terceiros).

## Borda e identidade

| Componente | Configuração |
| --- | --- |
| Azure Front Door Premium + WAF | Domínio único (`arqflow.<empresa>`). `/*` → Blob, `/bff/*` → BFF. WAF em modo Prevention, regras gerenciadas + rate limit por IP. Headers: CSP restritiva, HSTS, `X-Content-Type-Options`, `Referrer-Policy`, `frame-ancestors 'none'`. |
| Microsoft Entra ID | App registration do BFF (confidential client, certificado/federated credential, redirect URI exata) e app registration das APIs (scopes `demandas.read`, `demandas.write`, `custeio.read` e app roles Solicitante, Arquiteto, Gestor/PMO). Um registro por ambiente. |
| BFF | Authorization Code + PKCE; tokens no Cache de Sessão; cookie `__Host-` `HttpOnly; Secure; SameSite`; CSRF por header customizado; renovação com rotação do refresh token; logout local + `end_session_endpoint`. |
| Sensedia API Gateway | Plataforma compartilhada da empresa. Valida JWT (assinatura via JWKS em cache, `iss`, `aud`, `exp`, scopes), rate limit e quotas por API, roteamento para o ingress interno do AKS por mTLS. |

## Rede

VNet em Brazil South, com três zonas de disponibilidade. Subnets separadas para o AKS (`snet-aks`) e para os Private Endpoints (`snet-private-endpoints`). Nenhum serviço de dados tem acesso público habilitado. O AKS usa Azure CNI com Cilium para `NetworkPolicy` default-deny entre os namespaces de domínio ([ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md)), com exceções explícitas por fluxo conhecido. O BFF é exposto ao Front Door por um Private Link Service sobre o load balancer interno — não existe IP público de entrada no cluster.

## Compute — AKS

- AKS no tier Standard (SLA do control plane), node pools distribuídos nas três zonas para as cargas estáveis (BFF, Demanda/Cenário, Dashboard, Motor de Custeio).
- Node Auto Provisioning (baseado em Karpenter) para cargas em rajada (worker de integração EA, jobs de custeio), com nós Spot onde a carga tolera interrupção.
- Namespaces por domínio: `bff`, `demanda-cenario`, `custeio`, `integracao-ea`, `dashboard` e `platform`, cada um com `ResourceQuota`/`LimitRange`.
- Detalhes de RBAC, segurança de pod, autoscaling e entrega progressiva em [`kubernetes-boas-praticas.md`](kubernetes-boas-praticas.md).

## Dados

- **Azure Database for PostgreSQL — Flexible Server**, com alta disponibilidade zone-redundant e backup geo-redundante. Fonte de demandas, cenários, diagramas e estimativas. O Serviço de Dashboard usa um usuário de banco somente leitura.
- **Azure Cache for Redis — sessão:** tokens do BFF por sessão, criptografados pela aplicação, com TTL igual à expiração absoluta da sessão.
- **Azure Cache for Redis — preços:** cache do Motor de Custeio ([ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md)). Instância separada da de sessão, para que a política de despejo de um não afete o outro.
- **Azure Service Bus (Premium):** tópico de aprovação de cenário com assinatura do Serviço de Integração EA ([ADR-0003](../03-decisoes-arquiteturais/0003-sincronizacao-assincrona-com-bizzdesign.md)). Mensagens que esgotam as tentativas vão para a dead-letter queue, que dispara alerta no Azure Monitor — é isso que garante que "falha de sincronização nunca é silenciosa". O tier Premium é o que permite Private Endpoint (ver custo em [`estimativa-finops.md`](estimativa-finops.md)).

> **Atenção (Redis):** a Microsoft anunciou a substituição do Azure Cache for Redis pelo Azure Managed Redis. Confirmar o cronograma vigente antes da implantação e, se aplicável, usar o Azure Managed Redis com a mesma separação sessão/preços.

## Segurança e identidade de workload

- **Microsoft Entra Workload ID:** cada `ServiceAccount` usa uma managed identity própria e mínima. Só o BFF lê o certificado OAuth no Key Vault; só o worker de integração EA consome a assinatura do Service Bus e lê o token da BiZZdesign; só o CronJob de custeio escreve no cache de preços. Nenhum pod usa credencial estática.
- **Azure Key Vault:** certificado do cliente OAuth do BFF, chave de criptografia da sessão, credenciais de banco e token da BiZZdesign. Sincronizados para o cluster pelo External Secrets Operator (ou Secrets Store CSI Driver), nunca commitados.
- **Azure Container Registry (Premium):** Private Endpoint, tags imutáveis e scan de vulnerabilidade (Microsoft Defender for Containers). Imagem sem scan aprovado não é elegível para deploy (ver [`esteira-cicd.md`](esteira-cicd.md)).

## Observabilidade

Container Insights e Log Analytics para logs; Azure Monitor managed service for Prometheus e Azure Managed Grafana para métricas. O BFF gera o `traceparent` (W3C Trace Context) e ele é propagado pela Sensedia até os serviços, de modo que um login ou uma cotação pode ser seguido de ponta a ponta. Logs de segurança mínimos: login, logout, JWT rejeitado (401), falta de permissão (403), CSRF rejeitado e bloqueios do WAF. Tokens e cookies nunca são registrados.

## Estimativa de custo (FinOps)

Estimativa mensal desta arquitetura, com cenário otimizado e os trade-offs de cada alavanca, em [`estimativa-finops.md`](estimativa-finops.md).

## Infraestrutura como código

Terraform (provider `azurerm`), em módulos por camada (rede, AKS, dados, mensageria, identidade, borda), com state remoto em Azure Storage e lock por blob lease. `checkov`/`tfsec` roda como gate antes de qualquer `apply` (ver [`esteira-cicd.md`](esteira-cicd.md#pipeline-de-infraestrutura)). Ambientes (dev/hml/prd) têm state files e subscriptions separados.

## O que foi deliberadamente deixado de fora (por ora)

- **Service mesh completo (Istio):** avaliado e adiado. O mTLS já existe no trecho Sensedia → serviços; entre serviços internos, `NetworkPolicy` + Workload ID cobrem as garantias necessárias. Revisitar se o número de serviços crescer ou se for exigido mTLS em todos os saltos internos.
- **Multi-região ativa-ativa:** o RNF de disponibilidade (99,5%) é atendido com três zonas em uma região. O Front Door já é global, o que facilita uma segunda região se o RNF mudar.
