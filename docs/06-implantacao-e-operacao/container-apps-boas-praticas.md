# Azure Container Apps: organização e boas práticas

Este documento detalha como os serviços do ArqFlow são organizados e operados no Azure Container Apps — a continuação prática do [ADR-0012](../03-decisoes-arquiteturais/0012-azure-container-apps-como-plataforma-de-compute.md) (por que Container Apps e não AKS), do [ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md) (isolamento por domínio) e do [ADR-0014](../03-decisoes-arquiteturais/0014-entrega-continua-por-revisoes-do-container-apps.md) (entrega por revisões).

## Organização: ambiente, apps e jobs

Um **ambiente** por estágio (dev, hml, prd), cada um na própria subnet e subscription. Dentro do ambiente, **um Container App por domínio** — a unidade de Bulkhead que antes era o namespace:

| Recurso | Conteúdo | Observação |
| --- | --- | --- |
| `ca-bff` | BFF (cliente OAuth confidencial, token handler) | Único app que fala com o Entra ID para login e que recebe tráfego do Front Door |
| `ca-demanda-cenario` | Serviço de Demanda/Cenário | Publica eventos de aprovação no Service Bus |
| `ca-custeio` | Motor de Custeio | Lê o cache de preços |
| `ca-integracao-ea` | Adaptador BiZZdesign | Único app que consome a fila e fala com a BiZZdesign; sem ingress |
| `ca-dashboard` | Serviço de Dashboard | Somente leitura sobre o Azure SQL |
| `job-precos` | Atualização da tabela de preços (diária) | Único com escrita no cache de preços |
| `job-migracao-db` | Migrações do Azure SQL | Disparado pela esteira antes de liberar tráfego |

Cada app define CPU/memória por réplica e mínimo/máximo de réplicas. Uma rajada no worker de Integração EA escala só aquele app — não consome a capacidade de que o BFF precisa para responder ao usuário.

## Identidade de workload (Managed Identity)

Cada app e job tem uma **Managed Identity atribuída por usuário**, com permissão mínima: `AcrPull` no registry para todos; `Key Vault Secrets User` só nos segredos de que precisa; `Azure Service Bus Data Sender` para Demanda/Cenário e `Data Receiver` na assinatura para Integração EA; usuário de banco criado a partir da identidade (`FROM EXTERNAL PROVIDER`), com `db_datareader` para o Dashboard. Nenhuma credencial estática é configurada em variável de ambiente.

## Gestão de segredos

Segredos ficam no **Azure Key Vault** e o Container App os referencia pela URI do segredo, usando a Managed Identity. A plataforma busca e atualiza o valor; o Bicep e o repositório só contêm a referência, nunca o valor. A connection string do Azure SQL não tem senha (autenticação Entra).

## Rede e exposição

- Ingress **interno** para BFF, Demanda/Cenário, Custeio e Dashboard; **sem ingress** para o worker e os jobs.
- Nos serviços chamados pela Sensedia, o ingress exige **certificado de cliente** (mTLS).
- **Criptografia peer-to-peer** habilitada no ambiente: o tráfego entre apps é cifrado.
- Sem `NetworkPolicy`: o controle entre apps do mesmo ambiente é o tipo de ingress + NSG + revalidação do JWT em cada serviço. É mais fraco que default-deny em L3/L4 e está registrado como trade-off no ADR-0012.

## Postura de segurança das imagens

- Imagem final mínima (multi-stage, base *distroless* ou *slim*), processo como usuário não-root.
- Somente imagens do ACR corporativo, por **digest** — garantido por Azure Policy e pela esteira.
- Avaliação de vulnerabilidade no ACR pelo Defender for Containers; vulnerabilidade crítica sem exceção documentada bloqueia o deploy.

## Escala

- **HTTP** para BFF e serviços chamados pela Sensedia, por requisições concorrentes por réplica, calibradas com dados reais de uso (revisão periódica, não um número fixo do primeiro deploy).
- **KEDA (embutido)** para `ca-integracao-ea`: escala pelo número de mensagens na assinatura do Service Bus, de 0 a 5 réplicas, autenticando com a Managed Identity. CPU não reflete bem uma carga em rajada orientada a mensagens.
- **Mínimo de réplicas:** 2 nos serviços do caminho do usuário (sem cold start no login e tolerância a falha de zona); 0 no worker, que pode esperar alguns segundos.

## Disponibilidade

- Ambiente com **redundância de zona**: réplicas distribuídas entre as três zonas de Brazil South.
- Probes de `startup`, `readiness` e `liveness` calibradas para dependências reais. O serviço de Demanda/Cenário não fica "not ready" por lentidão na fila — a aprovação precisa continuar respondendo com a sincronização degradada (garantia do ADR-0003).
- Health probe do Front Door aponta para um endpoint leve do BFF.

## Entrega progressiva (revisões)

Apps em **modo de revisões múltiplas**. Cada deploy cria uma revisão nova e imutável, que entra com 0% de tráfego e um **rótulo** (ex.: `candidata`) para smoke tests. Em seguida, a esteira move o tráfego por pesos — 10% → 50% → 100% — e consulta o Application Insights entre os passos (taxa de 5xx e p95). Regressão devolve 100% à revisão anterior, em segundos e sem rebuild. Serviços menos críticos (Dashboard) vão direto a 100% após o smoke test. Detalhes em [`esteira-cicd.md`](esteira-cicd.md).

Como duas revisões convivem durante o canary, migrações do Azure SQL seguem **expand/contract**: primeiro adiciona-se o que a nova versão precisa sem quebrar a antiga; a remoção vem num deploy posterior.

## Operação

- **Acesso humano:** leitura por grupos do Entra ID; escrita em produção só por PIM (just-in-time, aprovado e auditado). Mudanças manuais no portal são exceção registrada e detectadas pelo `what-if` diário.
- **Console e logs:** `az containerapp logs` e o console da revisão para diagnóstico, nunca para corrigir em produção.

## Observabilidade

Toda métrica, log e trace carrega o nome do app/domínio (`cloud_RoleName`) — e o `tenant_id`, se a multi-tenancy voltar. Application Insights por serviço via agente OpenTelemetry gerenciado do ambiente; logs de sistema e console no Log Analytics; workbooks do Azure Monitor organizados por domínio, espelhando a organização dos apps.

## O que decidimos não fazer (ainda)

- **Dapr:** disponível no Container Apps, mas o ArqFlow já usa os SDKs do Service Bus e do Redis diretamente; adicionar Dapr seria uma abstração sem problema a resolver.
- **Perfis Dedicados:** só se o consumo constante tornar um nó mais barato que o pagamento por segundo (ver [`estimativa-finops.md`](estimativa-finops.md)).
