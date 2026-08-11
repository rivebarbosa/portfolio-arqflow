# ADR-0007: Entregar em produção via GitOps (Argo CD), não com a esteira de CI aplicando mudanças diretamente no cluster

**Status:** aceito
**Data:** 2026-03-06
**Decisores:** Arquitetura, Segurança

## Contexto

Com EKS definido (ADR-0005), falta decidir como uma mudança aprovada chega até o cluster. O modelo mais comum e mais simples é a esteira de CI (GitHub Actions) rodar `kubectl apply`/`helm upgrade` diretamente contra o cluster de produção ao final do pipeline. O ArqFlow lida com dados sensíveis de arquitetura corporativa de múltiplos clientes (ver [`contexto-negocio.md`](../01-discovery/contexto-negocio.md)) e o autor deste portfólio tem foco declarado em Zero Trust — a superfície de credenciais com poder de escrita em produção é, portanto, uma preocupação de primeira ordem, não um detalhe de implementação.

## Alternativas consideradas

### Opção A — Deploy direto (push-based): a esteira de CI aplica as mudanças no cluster ao final do pipeline
- Prós: fluxo linear e fácil de entender; menos componentes rodando no cluster.
- Contras: exige credenciais de longa duração com permissão de escrita em produção armazenadas na esteira de CI — ampliando a superfície de ataque exatamente do tipo de dado sensível que o produto promete proteger; nenhuma reconciliação automática se alguém alterar o cluster manualmente (drift silencioso); rollback depende de re-executar a pipeline com uma versão anterior, não de uma operação de Git.

### Opção B — GitOps (pull-based): a esteira de CI só publica a imagem e atualiza um repositório de manifests/Helm values; um controlador dentro do cluster (Argo CD) observa esse repositório e aplica as mudanças
- Prós: nenhuma credencial de escrita em produção sai do cluster — o controlador puxa a mudança, a esteira nunca recebe um kubeconfig de produção; reconciliação contínua corrige drift automaticamente; rollback é reverter um commit no repositório de manifests, auditável pelo histórico do Git; promoção entre ambientes (dev → staging → prod) fica explícita como um Pull Request entre diretórios/overlays de ambiente.
- Contras: mais um componente para operar dentro do cluster; adiciona uma etapa entre "a imagem foi publicada" e "a mudança está de fato em produção", exigindo observabilidade própria do estado de sincronização.

## Decisão

Adotar GitOps com Argo CD. A esteira de CI (GitHub Actions, ver [`esteira-cicd.md`](../06-implantacao-e-operacao/esteira-cicd.md)) é responsável apenas por build, teste, scan de segurança, publicação da imagem no ECR e atualização automatizada do repositório de manifests. O Argo CD, rodando dentro do próprio EKS, reconcilia o estado do cluster a partir desse repositório.

## Padrão arquitetural

**Reconciliation Loop** (o mesmo princípio de control loop nativo do Kubernetes — um controlador observa o estado desejado e corrige o estado real continuamente — aplicado um nível acima, ao próprio processo de deploy). Também é uma aplicação direta de **least privilege**: a esteira de CI, um sistema externo ao cluster, nunca precisa de uma credencial de escrita em produção para cumprir sua função.

## Justificativa

Dado que o próprio produto vende garantia de governança e segurança de arquitetura corporativa a seus clientes, seria inconsistente proteger dados em repouso (ADR-0001, criptografia — ver [`requisitos-nao-funcionais.md`](../02-requisitos/requisitos-nao-funcionais.md#segurança)) e ao mesmo tempo manter uma credencial de escrita irrestrita em produção guardada em segredo de CI. GitOps resolve isso estruturalmente, não por disciplina de processo.

## Consequências

- É necessário manter um repositório de manifests/Helm values separado (ou uma pasta clara dentro do monorepo) por ambiente, com o padrão *app-of-apps* do Argo CD organizando os namespaces definidos no ADR-0006.
- A equipe precisa de visibilidade sobre o próprio Argo CD (dashboard e alertas de falha de sincronização), além dos logs do GitHub Actions — um deploy "com sucesso" na esteira não significa, por si só, que o cluster já reflete a mudança.
- O tempo entre "mudança aprovada" e "mudança em produção" passa a depender também do intervalo de reconciliação do Argo CD, não apenas da duração do pipeline de CI.

## Revisitar quando

Se a equipe permanecer pequena e a sobrecarga de operar o Argo CD não se justificar frente ao ganho de segurança/auditoria, considerar um meio-termo: push-based, mas com credenciais de curta duração via federação OIDC entre GitHub Actions e uma IAM Role com permissão restrita e temporária — evita chave estática de longa duração sem exigir um controlador adicional no cluster.
