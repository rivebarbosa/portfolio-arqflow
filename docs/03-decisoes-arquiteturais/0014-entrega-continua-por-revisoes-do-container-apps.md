# ADR-0014: Entregar por revisões do Container Apps, com GitHub Actions (OIDC) e Bicep — não GitOps com Argo CD

**Status:** aceito
**Data:** 2026-11-02
**Decisores:** Arquitetura, Segurança
**Substitui:** [ADR-0007](0007-gitops-com-argocd-para-entrega-continua.md)

## Contexto

O [ADR-0007](0007-gitops-com-argocd-para-entrega-continua.md) escolheu GitOps com Argo CD porque, em Kubernetes, ele elimina a credencial de escrita em produção fora do cluster: o controlador puxa as mudanças. Com o [ADR-0012](0012-azure-container-apps-como-plataforma-de-compute.md), não existe mais cluster — o Container Apps é gerenciado pela API do Azure Resource Manager, e não há onde rodar um controlador pull-based. É preciso redefinir como uma imagem aprovada chega a produção, mantendo a stack Microsoft (GitHub, Entra ID, ACR, Bicep).

O próprio ADR-0007 já registrava o meio-termo para quando o Argo CD deixasse de se justificar: **push-based, com credenciais de curta duração via federação OIDC e permissão restrita e temporária**.

## Alternativas consideradas

### Opção A — Push com credencial estática (service principal com client secret no GitHub)
- Prós: simples, documentado em todo lugar.
- Contras: segredo de longa duração com escrita em produção guardado no CI — exatamente o risco que o ADR-0007 quis eliminar.

### Opção B — Azure DevOps Pipelines com service connection por workload identity federation
- Prós: stack Microsoft; aprovações e environments maduros.
- Contras: o código e a esteira de CI já estão no GitHub; separar CI e CD em duas ferramentas não traz ganho de segurança (OIDC é paridade entre as duas).

### Opção C — GitHub Actions com OIDC (workload identity federation do Entra ID), deploy de nova revisão no Container Apps, promoção por GitHub Environments e infraestrutura em Bicep
- Prós: nenhum segredo armazenado — o token do Entra ID é emitido por execução e expira em minutos; a identidade de deploy só tem `Container Apps Contributor` no resource group do ambiente e `AcrPush` no registry; a federação é restrita ao repositório **e** ao GitHub Environment (`repo:...:environment:prd`), então só um job aprovado consegue o token de produção; rollback em segundos movendo o tráfego para a revisão anterior; Bicep é a linguagem de IaC nativa do Azure, com `what-if` para revisar mudanças.
- Contras: perde a reconciliação contínua do Argo CD (drift não é corrigido sozinho); existe, por alguns minutos a cada deploy, uma credencial com escrita em produção fora do ambiente.

## Decisão

Adotar a Opção C:

- **CI (GitHub Actions):** build, testes, CodeQL, Microsoft Security DevOps (inclui Trivy e Checkov/Template Analyzer), push para o **Azure Container Registry** com tag imutável (SHA do commit).
- **CD:** a esteira cria uma **nova revisão** do Container App com a imagem por digest, sem tráfego (`modo de revisões múltiplas`). Smoke tests rodam no rótulo da revisão; em seguida o tráfego passa por pesos (10% → 50% → 100%) com verificação automática de erro e latência no Application Insights entre cada passo. Regressão devolve 100% à revisão anterior.
- **Promoção:** `dev` → `hml` → `prd` por GitHub Environments; `prd` exige aprovação manual e janela de deploy.
- **Infraestrutura:** **Bicep** (Azure Verified Modules), `what-if` em todo PR, `deployment` em produção só com aprovação. Um job agendado roda `what-if` diário em produção e abre issue se encontrar drift.
- **Postura mínima:** **Azure Policy** no escopo da subscription bloqueia, na criação, Container App com ingress externo, ACR/SQL/Key Vault com acesso público e imagens fora do ACR corporativo.

## Padrão arquitetural

**Push-based deployment com credenciais efêmeras** (least privilege no tempo, não só no escopo) e **Blue/Green/Canary por revisões imutáveis**: cada revisão é um snapshot imutável da configuração, e "voltar" é mover tráfego, não reconstruir.

## Justificativa

A garantia que importava no ADR-0007 era "nenhuma credencial de longa duração com escrita em produção". Com OIDC restrito ao Environment aprovado, essa garantia continua: não há segredo para vazar, e o token só existe durante um job que passou por aprovação. O que se perde é a correção automática de drift — compensada por Azure Policy (impede a configuração errada) e pelo `what-if` diário (detecta a que escapar).

## Consequências

- O repositório de manifests e o Argo CD deixam de existir; o estado desejado da aplicação passa a ser o Bicep + a tag da imagem registrada no deploy.
- Rollback de aplicação = mover tráfego para a revisão anterior (segundos). Rollback de infraestrutura = reverter o commit do Bicep e reaplicar.
- Alterações manuais no portal em produção são proibidas por RBAC (ninguém tem escrita permanente; acesso via PIM just-in-time) e detectadas pelo `what-if` diário.
- Migrações de banco (Azure SQL, [ADR-0013](0013-azure-sql-database-como-banco-relacional.md)) rodam como um Container Apps Job disparado pela esteira antes de liberar tráfego para a nova revisão, e precisam ser compatíveis com a versão anterior (expand/contract), já que as duas revisões convivem durante o canary.

## Revisitar quando

Se o ArqFlow voltar a AKS (gatilhos do ADR-0012), voltar a avaliar GitOps. Se drift em produção aparecer com frequência no `what-if` diário, endurecer Azure Policy para `deny` nos atributos afetados.
