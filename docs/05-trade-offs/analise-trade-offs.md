# Análise de Trade-offs

## Isolamento forte vs. simplicidade operacional (multi-tenancy)

**O que escolhemos:** schema por tenant (ADR-0001) em vez de linha compartilhada com `tenant_id`.

**O que ganhamos:** uma garantia de isolamento que não depende de disciplina perfeita da aplicação em cada consulta — relevante em um produto que vende justamente confiança sobre dados sensíveis de arquitetura corporativa.

**O que perdemos:** simplicidade de migração de schema (agora multiplicada pelo número de tenants) e de consultas administrativas cross-tenant. Esse é um trade-off consciente de "mais trabalho operacional agora" em troca de "menos risco de vazamento entre clientes depois" — o tipo de troca que vale a pena quando o próprio modelo de negócio (B2B, vendendo a times de segurança/arquitetura) valoriza essa garantia mais do que a conveniência operacional.

## Controle total vs. delegação a um componente especializado (identidade)

**O que escolhemos:** gateway de identidade federada (ADR-0002) em vez de integrar cada IdP diretamente no core do produto.

**O que ganhamos:** a complexidade de suportar N provedores de identidade não cresce linearmente com o código de negócio do ArqFlow — cresce na configuração de um componente especializado nisso.

**O que perdemos:** um ponto único de falha adicional (se o gateway cair, ninguém loga em nenhum tenant) e uma dependência de operar bem um componente de infraestrutura de segurança, que não é o core business do produto. Esse é o trade-off clássico de "build vs. delegar para um componente especializado" — aceito porque autenticação corporativa é um domínio maduro (protocolos padronizados), diferente de, por exemplo, o motor de custeio, que é uma diferenciação real do produto e por isso foi construído sob medida.

## Consistência forte vs. resiliência a um sistema externo (integração BiZZdesign)

**O que escolhemos:** sincronização assíncrona com retry (ADR-0003) em vez de chamada síncrona no momento da aprovação.

**O que ganhamos:** o fluxo crítico do produto (aprovar um cenário) nunca fica refém da disponibilidade de um sistema que não controlamos.

**O que perdemos:** simplicidade — "aprovado" e "sincronizado com a BiZZdesign" deixam de ser o mesmo evento, exigindo modelar e comunicar dois estados distintos ao usuário. Esse trade-off só é aceitável porque o discovery confirmou explicitamente que o time de EA do cliente já revisa manualmente essas sincronizações — se a expectativa fosse de sincronização instantânea e automática sem revisão humana, esse desenho precisaria ser reaberto.

## Precisão em tempo real vs. previsibilidade e velocidade (motor de custeio)

**O que escolhemos:** tabela de preços cacheada com defasagem de até 24h (ADR-0004) em vez de consulta síncrona às APIs de precificação de nuvem.

**O que ganhamos:** cotações instantâneas (RNF < 1s) e comparações de cenário deterministicamente reprodutíveis — dois cenários compostos no mesmo dia têm preços consistentes entre si.

**O que perdemos:** exatidão em tempo real, que nem sequer é o objetivo real deste componente (é uma ferramenta de decisão, não de faturamento). Esse é o trade-off mais claro dos quatro: a "precisão perfeita" seria estritamente pior para o caso de uso real, porque prejudicaria a comparação entre cenários ao introduzir variação de preço não relacionada a mudanças no próprio cenário.

## O padrão por trás desses trade-offs

Em três das quatro decisões (identidade, integração BiZZdesign, custeio), a escolha foi **não assumir que um sistema externo é tão confiável ou controlável quanto o próprio produto** — seja um IdP de cliente, a API da BiZZdesign, ou a API de precificação de uma nuvem. Essa é a lição central deste caso de estudo: arquitetura de um produto B2B que integra com terceiros passa mais tempo desenhando para a *falha e a variabilidade* desses terceiros do que desenhando o "caminho feliz" interno.
