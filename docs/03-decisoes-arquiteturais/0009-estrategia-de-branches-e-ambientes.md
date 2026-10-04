# ADR-0009: Estratégia de branches no Git, mapeada 1:1 aos três ambientes

**Status:** aceito
**Data:** 2026-09-21
**Decisores:** Arquitetura (Roberto Rivelino Barbosa)

## Contexto

O código real do ArqFlow (ver ADR-0008) precisa rodar em três ambientes — Desenvolvimento, Homologação e Produção — hospedados em um repositório GitHub privado. Faltava decidir COMO o Git organiza isso: quantas branches de longa duração existem, o que cada uma representa, e como uma mudança "viaja" de um ambiente para o outro. Este é exatamente o ponto que o Roberto pediu para ser explicado: "qual a melhor forma de se trabalhar com os três ambientes no GitHub e como definir as branches para que reflitam os deploys nos ambientes corretos".

Restrições relevantes:
- Time pequeno (hoje, uma pessoa) — um processo pensado para dezenas de desenvolvedores (ex.: GitFlow completo, com branches de release e de suporte) adiciona cerimônia sem benefício correspondente.
- Precisa ficar óbvio, só de olhar o nome da branch, qual ambiente ela representa — é um requisito pedagógico tanto quanto técnico.
- O pipeline de CI/CD (ver `.github/workflows/`) precisa de um gatilho determinístico por ambiente: "push/merge nesta branch" → "build e publica a imagem Docker deste ambiente".

## Alternativas consideradas

### Opção A — Trunk-based development: uma única branch (`main`), deploys controlados por feature flags e por qual commit foi marcado como "liberado" para cada ambiente
- Prós: é o modelo que escala melhor para times grandes e múltiplos deploys por dia; evita o custo de manter branches de longa duração sincronizadas.
- Contras: exige infraestrutura de feature flags e disciplina de deploy que não existem ainda neste projeto; para um único desenvolvedor aprendendo a stack, perde exatamente a vantagem pedagógica de "o nome da branch já me diz o ambiente"; mapear commit → ambiente sem branches fica menos visual no GitHub.

### Opção B — GitFlow completo (branches `develop`, `release/*`, `hotfix/*`, `main`, com tags de versão obrigatórias)
- Prós: modelo muito documentado; formaliza bem o processo de release.
- Contras: pensado para times maiores com ciclos de release formais; as branches `release/*` adicionam uma etapa extra que, com um ambiente de Homologação já cumprindo esse papel de "pré-produção", vira duplicação; mais cerimônia do que o projeto precisa agora.

### Opção C — GitFlow simplificado: três branches de longa duração, uma por ambiente (`develop` → Desenvolvimento, `homolog` → Homologação, `main` → Produção), com `feature/*` de curta duração e `hotfix/*` opcional
- Prós: mapeamento 1:1 entre branch e ambiente — direto de entender e de configurar no CI/CD; promoção entre ambientes é sempre uma Pull Request de uma branch de ambiente para a próxima (nunca um push direto), então toda mudança em Homologação e Produção fica visível e auditável no histórico de PRs; cada ambiente sempre reflete exatamente o conteúdo da sua branch — sem ambiguidade sobre "o que está publicado onde".
- Contras: exige manter as três branches razoavelmente sincronizadas (evitar `homolog` ou `main` divergirem de `develop` por muito tempo); mais pesado que trunk-based se o projeto crescer para múltiplos deploys por dia — aceitável no estágio atual.

## Decisão

Adotar a Opção C: três branches de longa duração, cada uma correspondendo a exatamente um ambiente, mais branches de curta duração para o trabalho do dia a dia.

```
feature/nome-da-feature ──PR──► develop ──PR──► homolog ──PR──► main
     (curta duração)      (Desenvolvimento)   (Homologação)   (Produção)

hotfix/nome-do-bug ──PR──► main, e depois replicado para homolog e develop
     (curta duração, só para correções urgentes em produção)
```

| Branch | Ambiente | Protegida? | Quem publica nela | O que dispara no CI/CD |
|---|---|---|---|---|
| `feature/*` | nenhum (só a máquina do desenvolvedor / `docker compose up`) | não | qualquer pessoa, a partir de `develop` | `ci.yml` roda ao abrir PR para `develop` |
| `develop` | Desenvolvimento | sim (exige PR + CI verde) | merges de `feature/*` | `ci.yml` |
| `homolog` | Homologação | sim (exige PR + CI verde) | merges de `develop` | `ci.yml` na PR, depois `deploy-homolog.yml` no merge |
| `main` | Produção | sim (exige PR + CI verde, idealmente aprovação manual) | merges de `homolog` (ou `hotfix/*`) | `ci.yml` na PR, depois `deploy-prod.yml` no merge |
| `hotfix/*` | nenhum diretamente | não | quem estiver corrigindo produção com urgência, a partir de `main` | `ci.yml` na PR |

### Fluxo do dia a dia

1. Todo trabalho novo começa em uma branch `feature/nome-descritivo`, criada a partir de `develop`.
2. Ao terminar, abre-se uma Pull Request `feature/... → develop`. O workflow `ci.yml` roda automaticamente (lint, testes, build de backend e frontend). Só é possível mesclar com o CI verde.
3. Quando um conjunto de mudanças em `develop` está pronto para validação mais formal, abre-se uma PR `develop → homolog`. Ao ser mesclada, o workflow `deploy-homolog.yml` builda as imagens Docker de produção do backend/frontend e publica no GitHub Container Registry (GHCR) com a tag `homolog`.
4. Depois de validado em Homologação, abre-se uma PR `homolog → main`. Ao ser mesclada, `deploy-prod.yml` publica as imagens com a tag `latest`/`prod-<sha>`.
5. Se um bug crítico aparecer em Produção antes da próxima rodada normal, cria-se uma `hotfix/nome-do-bug` a partir de `main`, corrige-se, PR direto para `main` — e, assim que possível, a mesma correção é replicada para `homolog` e `develop` (`git merge main` nessas branches, ou uma PR equivalente), para as três branches nunca divergirem silenciosamente.

## Padrão arquitetural

Isto é uma instância do padrão **Environment Branching** (branch de longa duração = ambiente), combinado com **proteção de branch + revisão obrigatória** como o único caminho de promoção entre ambientes — nenhum ambiente além do local de desenvolvimento aceita push direto.

## Justificativa

O critério decisivo foi o pedido explícito do Roberto por uma correspondência clara entre branch e ambiente, e pela didática do processo. A Opção C entrega isso com o menor número de conceitos novos possível (comparado a GitFlow completo), sem abrir mão da principal garantia de segurança: nenhuma mudança chega a Homologação ou Produção sem passar por uma Pull Request revisável e por CI verde.

## Consequências

- É preciso configurar, no GitHub (`Settings → Branches`), regras de proteção para `develop`, `homolog` e `main`: exigir Pull Request, exigir que o workflow `ci.yml` passe, e — pelo menos para `main` — considerar exigir 1 aprovação (mesmo sendo um projeto de portfólio, é a prática que se espera documentar).
- Os segredos de cada ambiente (`JWT_SECRET`, `DATABASE_URL` de homologação/produção) devem ser cadastrados como *GitHub Actions Secrets*, nunca commitados — os arquivos em `docs/environments/*.example` documentam os nomes esperados, não os valores reais.
- Enquanto a infraestrutura real de Homologação/Produção não for contratada (ver ADR-0008), os workflows de deploy publicam as imagens no GHCR mas não completam a última etapa (instalar a imagem em um servidor) — isso está marcado como TODO nos próprios arquivos de workflow.
- Se o time crescer significativamente ou a cadência de deploy aumentar muito, revisitar a favor de trunk-based (Opção A) para reduzir o custo de manter três branches sincronizadas.

## Revisitar quando

Se o número de pessoas contribuindo crescer a ponto de PRs entre `develop`/`homolog`/`main` virarem gargalo, ou se a frequência de deploy desejada passar a ser "várias vezes por dia" (cenário em que trunk-based com feature flags costuma compensar mais).
