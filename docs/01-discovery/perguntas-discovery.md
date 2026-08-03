# Perguntas de Discovery

## "Cada cliente vai ter sua própria instância, ou é tudo compartilhado?"

**Resposta:** Multi-tenant compartilhado — o time comercial não pode depender de provisionar infraestrutura nova a cada venda; onboarding precisa ser self-service.
**Impacto:** Essa resposta, sozinha, elimina a opção de "instância dedicada por cliente" e força uma decisão explícita sobre como isolar dados entre tenants dentro de uma infraestrutura compartilhada — ver [ADR-0001](../03-decisoes-arquiteturais/0001-multi-tenancy-schema-por-tenant.md).

## "Quantos provedores de identidade diferentes precisamos suportar no dia 1?"

**Resposta:** Não há uma lista fechada — os 3 clientes-piloto em conversa já usam Azure AD, Okta e Google Workspace, respectivamente, três provedores diferentes.
**Impacto:** Construir integração ponto-a-ponto com cada provedor não escalaria (o 4º cliente pode trazer um 4º provedor). Essa resposta motivou diretamente adotar um gateway de identidade compatível com OIDC/SAML como camada de abstração, em vez de integrar cada IdP diretamente no core do produto (ver [ADR-0002](../03-decisoes-arquiteturais/0002-federacao-de-identidade-via-gateway-oidc.md)).

## "A sincronização com a BiZZdesign precisa ser em tempo real?"

**Resposta:** Não. O time de EA do cliente revisa e ajusta manualmente os artefatos importados de qualquer fonte — uma janela de algumas horas é aceitável, desde que a sincronização não se perca silenciosamente.
**Impacto:** Isso abriu espaço para tratar a integração como assíncrona e resiliente a indisponibilidade temporária da API da BiZZdesign, em vez de exigir uma chamada síncrona no momento da aprovação do cenário (que travaria o fluxo do arquiteto caso a BiZZdesign estivesse fora do ar) — ver [ADR-0003](../03-decisoes-arquiteturais/0003-sincronizacao-assincrona-com-bizzdesign.md).

## "A cotação de custo (FinOps) precisa refletir o preço da nuvem naquele segundo exato?"

**Resposta:** Não — o time de FinOps trabalha com estimativas para decisão, não faturamento; uma defasagem de até 24h no preço-base é aceitável e, na prática, mais previsível para comparar cenários (preços de cloud mudam, e recotar um cenário antigo com preço diferente do dia da decisão gera confusão).
**Impacto:** Motivou desenhar o motor de custeio em torno de uma tabela de preços cacheada e atualizada periodicamente, em vez de chamar as APIs de precificação da AWS/Azure/GCP a cada cotação — ver [ADR-0004](../03-decisoes-arquiteturais/0004-motor-de-custeio-com-cache-de-precos.md).

## "O que acontece se dois cenários da mesma demanda usarem nuvens diferentes?"

**Resposta:** É um caso real e esperado — parte do valor do produto é justamente comparar, por exemplo, um cenário em AWS vs. um em Azure para a mesma demanda.
**Impacto:** O motor de custeio precisa normalizar categorias de recurso (computação, armazenamento, rede) de forma comparável entre provedores, não apenas somar valores por provedor isoladamente — requisito que aparece em [RF-04](../02-requisitos/requisitos-funcionais.md).

## Premissas assumidas (perguntas sem resposta definitiva no discovery)

- **Premissa:** os 3 clientes-piloto representam bem a variedade de provedores de identidade do mercado-alvo. *Se um cliente futuro usar um provedor fora do padrão OIDC/SAML, o gateway de identidade precisa ser reavaliado.*
- **Premissa:** a API pública da BiZZdesign permanece estável o suficiente para uma integração de sincronização periódica (não confirmado com um contrato de SLA da BiZZdesign). *Se a API mudar sem aviso com frequência, a estratégia de integração (hoje um adaptador único) pode precisar de versionamento mais explícito por cliente.*
