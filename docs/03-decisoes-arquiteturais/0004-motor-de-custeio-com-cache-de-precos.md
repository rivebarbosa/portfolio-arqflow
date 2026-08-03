# ADR-0004: Motor de custeio baseado em tabela de preços cacheada, não chamada síncrona às APIs de nuvem

**Status:** aceito
**Data:** 2026-02-14
**Decisores:** Arquitetura, FinOps (consultoria)

## Contexto

RF-03 exige que, ao montar um cenário, o arquiteto veja uma estimativa de custo por recurso quase instantaneamente. O discovery confirmou que uma defasagem de até 24h no preço-base é aceitável — e, mais importante, é *preferível* do ponto de vista de FinOps, porque recotar um cenário antigo com o preço do dia atual (que pode ter mudado) geraria confusão sobre por que o mesmo cenário "mudou de custo" sem nenhuma alteração (ver [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#a-cotação-de-custo-finops-precisa-refletir-o-preço-da-nuvem-naquele-segundo-exato)).

## Alternativas consideradas

### Opção A — Consultar a API de precificação da nuvem (AWS Price List API, Azure Retail Prices API, GCP Billing Catalog) a cada vez que um recurso é adicionado a um cenário
- Prós: preço sempre o mais atual possível.
- Contras: adiciona uma dependência de rede síncrona externa no caminho crítico de uma ação de UI (adicionar um recurso), violando o RNF de latência (< 1s); preços podem mudar entre o momento da cotação e o momento da aprovação, sem que nada tenha mudado no cenário — prejudicial à rastreabilidade que RF-04 (comparação de cenários) depende.

### Opção B — Sincronizar periodicamente (a cada 24h) uma tabela de preços normalizada localmente, e calcular toda estimativa localmente a partir dela
- Prós: cálculo de custo é local e rápido (sem chamada de rede síncrona); todo cenário registra a data de referência do preço usado, tornando comparações e auditorias determinísticas; falha temporária da API de uma nuvem não impede cotações (usa o último cache válido).
- Contras: existe uma defasagem inerente de até 24h em relação ao preço público mais recente — inaceitável para faturamento real, mas aceitável para estimativa de decisão (que é o caso de uso real, confirmado no discovery); exige um job de sincronização periódico e monitorado por provedor de nuvem.

## Decisão

Manter uma tabela de preços normalizada (por categoria de recurso e por provedor), atualizada por um job periódico que consulta as APIs públicas de precificação de cada nuvem a cada 24h. Toda estimativa de cenário é calculada localmente a partir dessa tabela, e cada estimativa registra a data de referência do preço utilizado.

## Padrão arquitetural

**Materialized View** (pré-computa e armazena localmente uma visão derivada de uma fonte externa cara/lenta de consultar) combinado com **Cache-Aside** (a leitura de uma estimativa de custo sempre acessa o cache local primeiro, nunca a fonte externa diretamente) — ambos catalogados no Azure Architecture Center. O job periódico de sincronização é uma instância de **Scheduled Batch / Polling Consumer**: em vez de reagir a um evento (não há webhook confiável e uniforme entre AWS/Azure/GCP para preços), o próprio ArqFlow puxa a atualização em intervalo fixo (24h).

## Justificativa

A resposta do discovery de que defasagem de até 24h é aceitável — e até desejável para consistência — é o que permite tirar a chamada de precificação do caminho síncrono da UI, atendendo ao RNF de latência sem sacrificar a utilidade da estimativa para o propósito real (decisão comparativa, não faturamento).

## Consequências

- É necessário operar e monitorar um job de sincronização por provedor de nuvem, incluindo tratamento de mudanças no formato de resposta de cada API externa (schemas de precificação mudam com o tempo).
- Toda estimativa de custo carrega uma data de referência explícita — é uma decisão de produto tanto quanto técnica, para deixar claro ao usuário que não é um valor de faturamento garantido.
- Se um provedor de nuvem mudar drasticamente sua estrutura de precificação (nova categoria de recurso, por exemplo), a tabela normalizada interna precisa ser atualizada manualmente antes que aquele tipo de recurso possa ser cotado corretamente.

## Revisitar quando

Se o produto evoluir para oferecer também *faturamento real* (não apenas estimativa de decisão) integrado à conta de nuvem do cliente, essa decisão precisa ser reaberta — faturamento real não pode depender de uma tabela com até 24h de defasagem.
