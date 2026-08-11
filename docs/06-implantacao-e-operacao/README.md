# Implantação e Operação

Esta seção documenta como o ArqFlow roda em produção: a infraestrutura AWS de referência, como o Kubernetes (Amazon EKS) é organizado e operado, e como uma mudança sai do commit até o cliente. É a continuação natural dos [ADRs 0005–0007](../03-decisoes-arquiteturais/), que tratam das decisões de orquestração, isolamento multi-tenant em compute e modelo de entrega contínua.

| Documento | O que você vai encontrar |
| --- | --- |
| [`arquitetura-aws.md`](arquitetura-aws.md) | Arquitetura de referência na AWS — rede, EKS, dados, mensageria, segurança, observabilidade — com diagrama |
| [`kubernetes-boas-praticas.md`](kubernetes-boas-praticas.md) | Como o cluster é organizado e operado: namespaces, RBAC, rede, segredos, autoscaling, entrega progressiva |
| [`esteira-cicd.md`](esteira-cicd.md) | A esteira de CI/CD: da branch ao deploy em produção, com gates de qualidade e segurança |

## Como esta seção se conecta ao resto do portfólio

- A infraestrutura implementa, sem contradizer, as decisões de dados e identidade já tomadas em [ADR-0001](../03-decisoes-arquiteturais/0001-multi-tenancy-schema-por-tenant.md) e [ADR-0002](../03-decisoes-arquiteturais/0002-federacao-de-identidade-via-gateway-oidc.md) — o isolamento de tenant continua sendo schema + identidade federada, não topologia de cluster (ver [ADR-0006](../03-decisoes-arquiteturais/0006-isolamento-multi-tenant-por-dominio-nao-por-tenant.md)).
- A mensageria assíncrona descrita em [ADR-0003](../03-decisoes-arquiteturais/0003-sincronizacao-assincrona-com-bizzdesign.md) (fila + retry para a sincronização com a BiZZdesign) se materializa aqui em SQS/SNS com DLQ e alarme operacional.
- O motor de custeio (ADR-0004) roda como um serviço com seu próprio namespace, e a atualização periódica da tabela de preços é um `CronJob` dentro do cluster.
