# Contexto de Negócio

## O problema

Times de arquitetura de soluções, hoje, gerenciam seu fluxo de trabalho de forma fragmentada: a demanda chega por e-mail ou ticket, os cenários técnicos são desenhados em apresentações soltas, a cotação de custo é feita em uma planilha à parte (geralmente sem envolvimento formal do time de FinOps), e o desenho de arquitetura vive em um repositório de diagramas desconectado de tudo isso. O resultado:

- **Falta de rastreabilidade**: ninguém consegue responder rapidamente "por que escolhemos o cenário B e não o A" seis meses depois.
- **Cotação de custo tardia ou inexistente**: o time de FinOps só descobre o custo real depois que a solução já está em produção, quando é tarde para influenciar a decisão.
- **Desenho de arquitetura desatualizado**: o diagrama aprovado no início do projeto raramente reflete o que foi implementado, porque não há gatilho para atualizá-lo.
- **Repositório de arquitetura corporativa (EA) desalinhado**: empresas que já usam uma ferramenta de EA (como BiZZdesign) para manter o mapa de capacidades/aplicações não têm essas decisões refletidas lá automaticamente.

## Objetivo de negócio

Oferecer uma ferramenta única onde o arquiteto de soluções conduz uma demanda do recebimento à aprovação: registra a demanda, monta cenários alternativos, cota cada cenário (com apoio de FinOps embutido), anexa o desenho de arquitetura, e — ao aprovar — sincroniza a decisão com o repositório de EA do cliente (BiZZdesign) automaticamente.

## Métricas de sucesso (o que define "funcionou")

| Métrica | Situação atual (fragmentada) | Meta pós-lançamento |
|---|---|---|
| Tempo entre abertura da demanda e cenário aprovado | Sem medição padronizada (estimado 3-6 semanas) | Visível em dashboard, com meta de redução de 30% no 2º trimestre de uso |
| Cotação de custo disponível antes da aprovação | Ocorre em <30% das demandas (segundo entrevistas) | 100% das demandas aprovadas têm cotação FinOps registrada antes da aprovação |
| Decisões sincronizadas com a ferramenta de EA do cliente | Manual, esporádico | Sincronização automática em até 24h após aprovação |
| Adoção por arquitetos ativos (30 dias) | N/A (produto novo) | 70% dos arquitetos convidados usam a ferramenta pelo menos 1x/semana |

## Restrições conhecidas desde o início

- **Modelo de negócio multi-tenant B2B**: cada cliente é uma empresa com múltiplos arquitetos; dados de uma empresa nunca podem vazar para outra — essa é a restrição mais dura do produto, mais até que qualquer requisito funcional.
- **Cada cliente já tem seu próprio provedor de identidade** (Azure AD, Okta, Google Workspace, etc.) e não vai aceitar gerenciar mais uma senha — SSO é requisito de entrada no mercado, não um "nice to have".
- **BiZZdesign é um sistema de terceiros que não controlamos**: a integração precisa sobreviver a indisponibilidade, mudança de API e diferenças de plano contratado por cada cliente.
- **Prazo**: primeira versão comercializável (MVP vendável) em 5 meses, para apresentar a 3 clientes-piloto já em conversa comercial.

## Fora de escopo (decidido no discovery, não depois)

- Execução automatizada da infraestrutura (a ferramenta cota e documenta, não faz o *deploy* do cenário aprovado).
- Integração com outras ferramentas de EA além da BiZZdesign nesta primeira versão (ex: Ardoq, LeanIX) — arquitetura da integração deve permitir adicionar outras depois, mas não é escopo do MVP.
- Aprovação orçamentária formal (workflow de budget/PO) — a ferramenta mostra o custo estimado, mas o processo de aprovação financeira formal continua fora da ferramenta nesta versão.
