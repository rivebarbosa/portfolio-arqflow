# ADR-0006: Isolar o cluster por domínio/serviço, não por tenant — a fronteira de tenant continua sendo o schema, não o namespace

**Status:** aceito
**Data:** 2026-03-04
**Decisores:** Arquitetura, Segurança

## Contexto

Com EKS definido como orquestrador (ADR-0005), é preciso decidir como o multi-tenancy do produto se traduz para a camada de compute. Existe uma tentação natural de espelhar a decisão de dados (schema por tenant, ADR-0001) diretamente na camada de Kubernetes — um namespace por tenant. O RNF de onboarding self-service em até 10 minutos e a premissa, já validada no discovery, de que o ArqFlow não roda instância dedicada por cliente (ver [`perguntas-discovery.md`](../01-discovery/perguntas-discovery.md#cada-cliente-vai-ter-sua-própria-instância-ou-é-tudo-compartilhado)) tornam essa pergunta concreta e não apenas teórica.

## Alternativas consideradas

### Opção A — Namespace por tenant (replicar cada serviço para cada cliente)
- Prós: isolamento de compute forte, blast radius de uma falha limitado a um tenant.
- Contras: contradiz diretamente a razão pela qual o ADR-0001 rejeitou banco dedicado por tenant — o mesmo argumento de custo/operação que não escala para banco não escala para compute; onboarding deixaria de ser configuração (novo schema) e passaria a exigir provisionamento de infraestrutura nova a cada cliente; rollout de uma correção de bug ou nova versão precisaria ser coordenado em centenas de namespaces à medida que a base de clientes cresce.

### Opção B — Namespace por domínio/serviço, compute compartilhado entre tenants, isolamento de dados via schema (ADR-0001) e de identidade via token federado (ADR-0002)
- Prós: consistente com o modelo de dados já decidido; onboarding de um novo tenant continua sendo configuração, não deploy; cada domínio (`identidade`, `demanda-cenario`, `custeio`, `integracao-ea`, `dashboard`) ganha seu próprio namespace com `ResourceQuota` e `NetworkPolicy` próprios, limitando o impacto de um domínio problemático sobre os demais.
- Contras: um bug de propagação de contexto de tenant na camada de aplicação (ex: esquecer de resolver o schema correto a partir do token) tem impacto potencialmente mais visível, já que o compute é compartilhado; "vizinho ruidoso" entre tenants dentro do mesmo serviço é possível se `requests`/`limits` não forem bem calibrados.

## Decisão

O cluster é organizado por domínio de negócio, não por tenant: um namespace por serviço (`identidade`, `demanda-cenario`, `custeio`, `integracao-ea`, `dashboard`), cada um compartilhado por todos os tenants. A fronteira de isolamento entre clientes continua sendo o schema PostgreSQL (ADR-0001) e o token de identidade federado (ADR-0002) — não a topologia do cluster.

## Padrão arquitetural

**Bulkhead Pattern**, aplicado na fronteira de domínio em vez de na fronteira de tenant: cada namespace tem `ResourceQuota` e `LimitRange` próprios para que uma pressão de carga no worker de integração EA (naturalmente em rajada, por natureza do ADR-0003) não consuma recursos que o serviço de identidade — que precisa responder rápido para todo login de qualquer tenant — dependa. É o mesmo princípio geral já estabelecido em [`analise-trade-offs.md`](../05-trade-offs/analise-trade-offs.md) ("não assumir que um sistema é tão confiável ou controlável quanto o próprio produto"), aqui aplicado entre domínios internos, não apenas contra sistemas de terceiros.

## Justificativa

Tratar dados e compute como se precisassem, por padrão, da mesma estratégia de isolamento seria confundir "isolamento multi-tenant" com "réplica de infraestrutura por cliente". O que o cliente enterprise compra — e o que o time de Segurança/IAM audita — é a garantia de que seus dados nunca vazam para outro tenant (ADR-0001) e de que só entra quem o próprio IdP do cliente autenticou (ADR-0002). Nenhuma dessas garantias exige namespace dedicado; exige, sim, que a fronteira que realmente importa (schema + identidade) seja auditável e testável — o que já é o caso.

## Consequências

- `requests`/`limits` de CPU/memória por deployment precisam ser calibrados por domínio e revisados com base em uso real, não estimados uma única vez — é a principal mitigação ao risco de vizinho ruidoso entre tenants dentro do mesmo serviço.
- `NetworkPolicy` default-deny entre namespaces de domínio, com exceções explícitas por fluxo conhecido (ex: `demanda-cenario` pode publicar no barramento de eventos consumido por `integracao-ea`; `dashboard` só pode ler do banco, nunca escrever).
- Toda métrica e log precisa carregar o identificador de tenant como atributo/label — já que o isolamento não vem da topologia do cluster, a observabilidade é o que permite investigar um incidente específico de um cliente em compute compartilhado (ver [`kubernetes-boas-praticas.md`](../06-implantacao-e-operacao/kubernetes-boas-praticas.md#observabilidade)).
- Um bug de propagação de contexto de tenant é, por desenho, um incidente de segurança de dados (mitigado pelo schema, ADR-0001) e não deveria nunca depender da topologia de namespaces como segunda linha de defesa real — a segunda linha de defesa é teste automatizado, não infraestrutura.

## Revisitar quando

Se um cliente enterprise específico exigir contratualmente isolamento físico de compute (não apenas de dados) — nesse caso, o próximo passo é um node group dedicado para aquele tenant dentro do mesmo cluster (via `taint`/`toleration` e `nodeSelector`), o que resolve a exigência sem replicar a topologia de namespaces para todos os demais clientes. Cluster dedicado só entraria em consideração se nem isso for suficiente.
