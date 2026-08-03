# Requisitos Funcionais

Cada requisito abaixo nasceu de uma resposta específica do discovery (ver [`01-discovery/`](../01-discovery/)).

## RF-01 — Registro de demanda

O arquiteto (ou solicitante de negócio) deve conseguir registrar uma nova demanda, com descrição, área solicitante, prioridade e prazo desejado.

- **Critério de aceite:** toda demanda criada recebe um identificador único, fica visível no dashboard do tenant, e pode ser associada a um ou mais cenários.
- **Origem:** [contexto-negocio.md](../01-discovery/contexto-negocio.md#o-problema)

## RF-02 — Criação de cenários alternativos

Para uma demanda, o arquiteto deve conseguir criar múltiplos cenários de solução (abordagens técnicas distintas), cada um independente para fins de cotação e diagrama.

- **Critério de aceite:** uma demanda pode ter N cenários; cada cenário tem seu próprio status (rascunho, em cotação, aprovado, descartado) independente dos demais.
- **Origem:** [contexto-negocio.md](../01-discovery/contexto-negocio.md#objetivo-de-negócio)

## RF-03 — Cotação de custo (FinOps) por cenário

Cada cenário deve permitir compor uma lista de recursos (computação, armazenamento, rede, licenciamento) e gerar uma estimativa de custo mensal/anual.

- **Critério de aceite:** ao adicionar um recurso a um cenário, o sistema retorna uma estimativa de custo com base na tabela de preços cacheada (ver [RNF](requisitos-nao-funcionais.md)), exibindo a data de referência do preço usado.
- **Origem:** [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#a-cotação-de-custo-finops-precisa-refletir-o-preço-da-nuvem-naquele-segundo-exato)

## RF-04 — Comparação de cenários entre provedores de nuvem

O sistema deve permitir comparar dois ou mais cenários da mesma demanda lado a lado, mesmo que usem provedores de nuvem diferentes, normalizando categorias de recurso.

- **Critério de aceite:** a tela de comparação agrupa custo por categoria normalizada (ex: "computação"), não apenas por nome de serviço específico do provedor.
- **Origem:** [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#o-que-acontece-se-dois-cenários-da-mesma-demanda-usarem-nuvens-diferentes)

## RF-05 — Anexação de desenho de arquitetura

Cada cenário deve permitir anexar (ou desenhar diretamente, via editor embutido) um diagrama de arquitetura, versionado junto ao cenário.

- **Critério de aceite:** um cenário sem diagrama pode ser salvo como rascunho, mas não pode ser movido para "aprovado" sem pelo menos um diagrama anexado.
- **Origem:** [contexto-negocio.md](../01-discovery/contexto-negocio.md#o-problema) (desenho desatualizado/desconectado)

## RF-06 — Aprovação de cenário e sincronização com BiZZdesign

Ao aprovar um cenário, o sistema deve registrar a decisão e disparar a sincronização dos artefatos relevantes (diagrama, descrição da solução) com o repositório de EA do cliente na BiZZdesign.

- **Critério de aceite:** a aprovação é instantânea do ponto de vista do arquiteto (não espera a BiZZdesign responder); o status de sincronização (pendente, sincronizado, falhou) fica visível separadamente, com nova tentativa automática em caso de falha.
- **Origem:** [perguntas-discovery.md](../01-discovery/perguntas-discovery.md#a-sincronização-com-a-bizzdesign-precisa-ser-em-tempo-real)

## RF-07 — Autenticação via SSO corporativo

Usuários devem acessar o ArqFlow exclusivamente através do provedor de identidade da própria empresa (tenant), sem necessidade de senha própria no ArqFlow.

- **Critério de aceite:** o login redireciona para o IdP configurado pelo tenant (Azure AD, Okta, Google Workspace ou qualquer provedor OIDC/SAML compatível); um usuário nunca vê uma tela de "criar senha" do ArqFlow.
- **Origem:** [stakeholders.md](../01-discovery/stakeholders.md) (Time de Segurança/IAM do cliente)

## RF-08 — Dashboard de andamento de projetos

Gestores devem visualizar, por tenant, quantas demandas existem por status, tempo médio em cada fase, e custo total aprovado vs. estimado.

- **Critério de aceite:** o dashboard reflete o estado mais recente disponível (tolerância de atraso de poucos minutos é aceitável — ver RNF de atualização de dashboard), filtrável por área solicitante e período.
- **Origem:** [contexto-negocio.md](../01-discovery/contexto-negocio.md#métricas-de-sucesso-o-que-define-funcionou)

---

**Fora de escopo explícito** (ver [contexto-negocio.md](../01-discovery/contexto-negocio.md#fora-de-escopo-decidido-no-discovery-não-depois)): execução automatizada do deploy do cenário aprovado, integração com outras ferramentas de EA além da BiZZdesign nesta versão, workflow formal de aprovação orçamentária.
