# Implantação e Operação

Esta seção documenta como o ArqFlow roda em produção: a infraestrutura Azure de referência, como o Kubernetes (AKS) é organizado e operado, e como uma mudança sai do commit até o usuário. Continua os [ADRs 0006, 0007, 0010 e 0011](../03-decisoes-arquiteturais/), que tratam de isolamento em compute, entrega contínua, identidade (BFF + Entra ID + Sensedia) e da plataforma Azure.

| Documento | O que você vai encontrar |
| --- | --- |
| [`arquitetura-azure.md`](arquitetura-azure.md) | Arquitetura de referência no Azure — borda, identidade, rede, AKS, dados, segurança, observabilidade — com diagrama |
| [`kubernetes-boas-praticas.md`](kubernetes-boas-praticas.md) | Como o cluster é organizado e operado: namespaces, RBAC, rede, segredos, autoscaling, entrega progressiva |
| [`esteira-cicd.md`](esteira-cicd.md) | A esteira de CI/CD: da branch ao deploy em produção, com gates de qualidade e segurança |
| [`estimativa-finops.md`](estimativa-finops.md) | Estimativa de custo mensal (baseline e otimizado), com os trade-offs de cada alavanca |

## Como esta seção se conecta ao resto do portfólio

- A borda e a identidade seguem a arquitetura de referência SPA + BFF + OAuth 2.0/JWT ([ADR-0010](../03-decisoes-arquiteturais/0010-bff-como-token-handler-com-entra-id-e-sensedia.md)): Front Door + WAF, BFF como token handler, Entra ID como IdP e Sensedia como API Gateway.
- A mensageria assíncrona do [ADR-0003](../03-decisoes-arquiteturais/0003-sincronizacao-assincrona-com-bizzdesign.md) (fila + retry para a BiZZdesign) se materializa em Azure Service Bus, com dead-letter queue e alerta no Azure Monitor.
- O motor de custeio ([ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md)) roda em seu próprio namespace, e a atualização da tabela de preços é um `CronJob` dentro do cluster, escrevendo num Redis separado do Redis de sessão do BFF.
