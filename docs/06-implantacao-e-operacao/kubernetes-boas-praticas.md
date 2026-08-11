# Kubernetes (EKS): organização e boas práticas

Este documento detalha como o cluster EKS é organizado e operado no dia a dia — a continuação prática dos [ADR-0005](../03-decisoes-arquiteturais/0005-eks-como-orquestrador-de-containers.md) (por que EKS), [ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md) (isolamento por domínio, não por tenant) e [ADR-0007](../03-decisoes-arquiteturais/0007-gitops-com-argocd-para-entrega-continua.md) (GitOps).

## Organização de namespaces

| Namespace | Conteúdo | Observação |
| --- | --- | --- |
| `identidade` | Gateway de Identidade (broker OIDC/SAML) | Único namespace autorizado a falar com IdPs externos |
| `demanda-cenario` | Serviço de Demanda/Cenário | Publica eventos de aprovação no barramento |
| `custeio` | Motor de Custeio + CronJob de atualização de preços | Único namespace com acesso de escrita ao cache de preços |
| `integracao-ea` | Adaptador BiZZdesign | Único namespace autorizado a consumir a fila e a falar com a BiZZdesign |
| `dashboard` | Serviço de Dashboard | Somente leitura sobre o banco |
| `platform` | Argo CD, External Secrets Operator, Karpenter, stack de observabilidade | Componentes de plataforma, não de domínio de negócio |

Cada namespace de domínio tem `ResourceQuota` e `LimitRange` próprios — a aplicação prática do Bulkhead Pattern descrito no ADR-0006: uma rajada no worker de integração EA não deveria conseguir consumir recursos que o serviço de identidade precisa para responder a um login de qualquer tenant.

## RBAC de mínimo privilégio

- Nenhuma pessoa ou pipeline de CI tem `cluster-admin` de rotina. Acesso de escrita em produção passa pelo Argo CD (ADR-0007), não por `kubectl` direto de um humano ou de uma esteira.
- `Role`/`RoleBinding` são escopados por namespace: o time responsável pelo Motor de Custeio não tem permissão de leitura/escrita sobre o namespace `identidade`, e vice-versa.
- Acesso de emergência ("break-glass") é uma role separada, com uso auditado e alertado — nunca uma credencial permanente de uso corrente.

## Isolamento de rede

`NetworkPolicy` default-deny em todos os namespaces de domínio, com liberação explícita por fluxo conhecido: `demanda-cenario → SNS` (publicação), `SQS → integracao-ea` (consumo), `custeio → ElastiCache`, `dashboard → RDS` (somente leitura, reforçado também no nível de usuário de banco). Qualquer fluxo não listado é bloqueado por padrão — a política nomeia o que é permitido, não o que é proibido.

## Identidade de workload (IRSA)

Cada `ServiceAccount` assume uma IAM Role própria via IRSA, com permissão mínima para a função daquele serviço (ex: a `ServiceAccount` de `integracao-ea` só pode consumir da fila SQS específica e ler o segredo da API da BiZZdesign no Secrets Manager). Nenhuma credencial AWS estática é distribuída como `Secret` do Kubernetes — eliminando uma classe inteira de risco (chave vazada em manifest, imagem ou log).

## Gestão de segredos

External Secrets Operator sincroniza segredos do AWS Secrets Manager para `Secret` nativos do Kubernetes, sob demanda e com rotação automática refletida no cluster. Segredo nunca é commitado em Helm values ou manifest versionado — o repositório de manifests do GitOps (ADR-0007) referencia o segredo pelo nome, nunca pelo valor.

## Postura de segurança de pods

Pod Security Admission em modo `restricted` para todos os namespaces de domínio: containers rodam como usuário não-root, sistema de arquivos raiz somente leitura, capabilities do Linux reduzidas ao mínimo necessário. Imagem sem scan de vulnerabilidade aprovado no ECR não passa no *admission* — reforçado também como gate na esteira (ver [`esteira-cicd.md`](esteira-cicd.md)).

## Requests, limits e autoscaling

- `requests`/`limits` calibrados por domínio com base em uso real observado, não estimados uma única vez no primeiro deploy — revisão periódica é parte do processo, não uma exceção.
- **HPA** orientado a métricas de negócio quando fizer sentido, não só CPU: o worker de `integracao-ea`, por exemplo, escala pela profundidade da fila SQS (via KEDA ou métrica customizada no Cloudwatch Adapter), não por utilização de CPU — que não reflete bem uma carga em rajada orientada a mensagens.
- **Karpenter** para autoscaling de nós, com consolidação automática — evita pagar por capacidade ociosa fora de horário comercial (o produto é usado majoritariamente em horário de trabalho dos clientes, conforme o RNF de disponibilidade já assume).

## Disponibilidade

- `PodDisruptionBudget` em todo serviço de domínio, réplicas distribuídas entre AZs.
- Probes de `readiness`/`liveness`/`startup` calibradas para refletir dependências reais — por exemplo, o serviço de Demanda/Cenário não deve reportar "not ready" por uma lentidão momentânea na fila de eventos, já que a aprovação de um cenário precisa continuar respondendo mesmo se a sincronização assíncrona downstream estiver degradada (é exatamente a garantia que o ADR-0003 promete).

## Entrega progressiva

Argo Rollouts para canary/blue-green nos serviços mais críticos para o RNF de disponibilidade (Demanda/Cenário, Motor de Custeio): uma nova versão recebe uma fração pequena do tráfego, métricas de erro/latência são avaliadas automaticamente, e o rollout avança ou sofre rollback automático sem intervenção manual em caso de regressão. Serviços menos críticos (Dashboard) usam rolling update padrão.

## GitOps como modelo operacional

Detalhado no [ADR-0007](../03-decisoes-arquiteturais/0007-gitops-com-argocd-para-entrega-continua.md): o cluster nunca recebe uma alteração aplicada diretamente por uma pessoa ou pela esteira de CI em condições normais de operação — o Argo CD reconcilia a partir de um repositório de manifests, o que também funciona como proteção contra *drift* de configuração.

## Observabilidade

Toda métrica, log e trace carrega `tenant_id` (quando aplicável) e `namespace`/domínio como atributo padronizado — é o que permite investigar um incidente específico de um cliente em um ambiente de compute compartilhado por desenho (ADR-0006). Prometheus (via Amazon Managed Prometheus) para métricas, Fluent Bit para logs, com dashboards no Grafana organizados por domínio, espelhando a organização de namespaces.

## O que decidimos não fazer (ainda)

Service mesh completo (Istio/App Mesh) foi avaliado e adiado — ver justificativa em [`arquitetura-aws.md`](arquitetura-aws.md#o-que-foi-deliberadamente-deixado-de-fora-por-ora). Documentar essa decisão explicitamente evita que ela pareça uma lacuna não percebida.
