# ADR-0003: Sincronizar com a BiZZdesign de forma assíncrona, nunca bloquear a aprovação do arquiteto

**Status:** aceito
**Data:** 2026-02-10
**Decisores:** Arquitetura

## Contexto

RF-06 exige que, ao aprovar um cenário, os artefatos relevantes sejam sincronizados com o repositório de EA do cliente na BiZZdesign. O discovery confirmou que essa sincronização não precisa ser instantânea (o time de EA do cliente revisa manualmente de qualquer forma) e que diferentes clientes podem estar em planos de API diferentes da BiZZdesign (alguns com webhook, outros só com polling) — ver [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#a-sincronização-com-a-bizzdesign-precisa-ser-em-tempo-real).

## Alternativas consideradas

### Opção A — Chamar a API da BiZZdesign de forma síncrona no momento da aprovação
- Prós: mais simples de implementar; o status "sincronizado" fica confirmado no mesmo instante da aprovação.
- Contras: a aprovação do arquiteto — uma ação de negócio interna ao ArqFlow — passaria a depender da disponibilidade de um sistema de terceiros que não controlamos; uma indisponibilidade ou lentidão da BiZZdesign travaria o fluxo crítico do produto.

### Opção B — Registrar a aprovação imediatamente e processar a sincronização de forma assíncrona (fila + adaptador de integração), com reprocessamento automático em caso de falha
- Prós: a aprovação do arquiteto nunca depende da BiZZdesign estar disponível; falhas de sincronização são tratadas como esperadas (reprocessamento), não exceções catastróficas; o adaptador pode ser ajustado por tenant (polling vs. webhook) sem mudar o core do produto.
- Contras: existe uma janela de tempo em que a aprovação já aconteceu mas a sincronização ainda está pendente — precisa ser visível ao usuário para não parecer que "sumiu"; exige lógica de retry com backoff e um limite antes de marcar como falha permanente e alertar.

## Decisão

A aprovação de um cenário é uma transação local do ArqFlow, concluída imediatamente. A sincronização com a BiZZdesign acontece de forma assíncrona via fila, com um adaptador de integração dedicado por tenant (compatível com o plano de API que aquele cliente tem), e political de retry automática dentro do prazo de 24h definido no RNF.

## Padrão arquitetural

**Queue-Based Load Leveling** combinado com **Retry Pattern** (ambos catalogados no Azure Architecture Center como padrões de integração em nuvem): a fila absorve a variação de disponibilidade/latência da BiZZdesign, e a política de retry com backoff trata falhas temporárias como esperadas, não excepcionais. O adaptador de integração por tenant é, novamente, um **Anti-Corruption Layer** — traduz o modelo interno do ArqFlow (cenário aprovado) para o esquema externo da BiZZdesign, isolando o core do produto de mudanças na API de terceiros. A interface genérica de "exportar artefato para EA" por trás da qual o adaptador vive é um **Strategy Pattern** (GoF): permite trocar/adicionar uma implementação (BiZZdesign hoje, Ardoq ou LeanIX amanhã) sem alterar o fluxo de aprovação que a consome.

## Justificativa

Tratar um sistema de terceiros que não controlamos como se fosse tão confiável quanto o próprio ArqFlow seria ignorar a natureza real da integração — a BiZZdesign pode ficar indisponível, mudar rate limits, ou ter latência variável, e nada disso deveria impedir um arquiteto de aprovar um cenário dentro do próprio produto.

## Consequências

- É necessário expor explicitamente o status de sincronização (pendente/sincronizado/falhou) na interface — esconder essa informação criaria a falsa impressão de que a integração é instantânea e sempre bem-sucedida.
- O adaptador de integração precisa ser desenhado por trás de uma interface genérica de "exportar artefato para EA", para permitir, no futuro, adicionar outra ferramenta de EA (Ardoq, LeanIX) sem reescrever o fluxo de aprovação.
- Falhas recorrentes de sincronização para um tenant específico precisam gerar alerta operacional — não é aceitável que uma integração fique silenciosamente quebrada por semanas.

## Revisitar quando

Se a BiZZdesign (ou uma futura integração de EA) oferecer garantias de disponibilidade e latência fortes o suficiente via contrato (SLA formal), pode valer a pena reavaliar uma opção de sincronização "quase síncrona" (poucos segundos) como upsell para clientes que queiram essa garantia — mas isso seria uma opção adicional, não substituiria o modelo assíncrono como padrão.
