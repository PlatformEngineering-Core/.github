# Setup da plataforma do zero

Este guia define a ordem de montagem da plataforma. Os scripts, contratos de
secrets e procedimentos completos continuam nos repositorios de cada componente.
Nao e um script de provisionamento unico nem uma confirmacao de recursos no Azure.

## Ordem recomendada

**Base externa -> gh-actions e templates -> gh-runners -> platform-enablement ->
OTel-Collector -> RavenDB e iDPw-backend -> iDPw -> primeira aplicacao.**

O primeiro projeto a colocar em funcionamento como capacidade de execucao e o
`gh-runners`, depois de preparar a base externa e publicar as dependencias de CI/CD.
O `platform-enablement` precisa do runner `platform`; o backend precisa dos estados
e workflows da plataforma; o portal precisa do backend para funcionar integralmente.
O Collector e o RavenDB podem ser preparados em paralelo apos seus pre-requisitos.

| Etapa | Componente | Depende de | Resultado necessario para avancar |
| --- | --- | --- | --- |
| 0 | GitHub, Azure e servicos externos | Acessos administrativos aprovados | Organizacoes, repositorios, subscription, topologia e credenciais definidos |
| 1 | Base Azure da plataforma | Etapa 0 | RG, Log Analytics, ACA Environment e storage de estados acessiveis |
| 2 | gh-actions e templates | Etapa 0 | Biblioteca publicada, mirror Apps sincronizado e templates acessiveis |
| 3 | gh-runners | Etapas 1 e 2 | Imagens publicadas e Jobs com labels, autenticacao e scaler coerentes |
| 4 | platform-enablement | Etapas 1 a 3 | Identidade, Environment da company, secrets e workflows prontos |
| 5 | OTel-Collector | Etapas 1 e 4, Grafana Cloud | Collector implantado e endpoint/bearer central cadastrados |
| 6 | RavenDB e iDPw-backend | Etapas 1, 2 e 4, GitHub App | API e scheduler operacionais, persistencia e leituras autorizadas |
| 7 | iDPw | Etapa 6, identidades de login | Portal autenticado e integrado ao backend |
| 8 | Aplicacao piloto | Etapas anteriores | Create, CI/CD, runtime, estados e telemetria verificados |

## 0. Definir organizacoes, acessos e destinos

- Disponibilize os repositorios Core e a organizacao de aplicacoes, normalmente
  `PlatformEngineering-Core` e `PlatformEngineering-Apps`. Habilite Actions e
  acesso aos workflows reutilizaveis conforme a visibilidade dos repositorios.
- Defina tenant, subscription, regiao, nomes de recursos, company e organizacao
  destino. Defaults e nomes do ambiente atual sao exemplos, nao recursos que
  existem automaticamente em uma instalacao nova.
- Tenha Git, PowerShell, Azure CLI e GitHub CLI no posto do operador. Os scripts
  Bash exigem ambiente compativel; build local de imagem exige Docker funcional.
  O caminho de bootstrap por GitHub-hosted runners evita exigir Docker local.
- Prepare as credenciais de runner Core e Apps, acesso ao GHCR, automacao Terraform
  e integracoes de seguranca exigidas pelas stacks. Consulte o
  [mapa de credenciais](../README.md#mapa-de-tokens-github-pats).
- Planeje RavenDB e Grafana Cloud. Sao servicos externos: publicar o backend nao
  cria o banco RavenDB, e publicar o Collector nao cria a stack Grafana.

Use nomes exatos de GitHub Environments. Nao troque silenciosamente uma company
com hifens por outra sem hifens. Nomes de secrets podem ser documentados, valores
nao: use os prompts seguros dos scripts ou o cadastro protegido do GitHub.

## 1. Preparar a base Azure da plataforma

Antes dos Jobs e dos deploys, prepare por um processo aprovado:

- Resource Group de operacao da plataforma.
- Log Analytics Workspace e Container Apps Environment associado a ele.
- Storage Account e container privado para estados e planos Terraform.
- Acesso de rede, providers Azure, permissoes e capacidade necessarios.

No ambiente de referencia aparecem `rg-platform-engineering`,
`law-platformengineering-core`, `cae-platformengineering` e o container
`projects-states`. Escolha e confira os destinos reais antes de executar scripts.
O nome da conta de storage tambem precisa ser coerente nos workflows, no backend
e nas configuracoes do Collector; nao basta alterar uma unica variable.

Esta etapa e um pre-requisito operacional, nao um bootstrap central automatizado
por este guia. O deploy do backend menciona a infra compartilhada da area platform,
mas apenas verifica o ACA Environment. Se a base for gerenciada por Terraform,
defina antes a raiz, o state e a identidade responsaveis; nao crie uma segunda
gestao dos mesmos recursos nem tente iniciar um runner em um environment ausente.

**Limite importante:** o script de Jobs dos runners cria/atualiza os Jobs, nao
substitui o provisionamento dessa base. O Terraform do Collector reutiliza RG,
environment e workspace existentes. Nao dependa do primeiro create de uma app
para criar a infraestrutura que o proprio runner precisa para executar o create.

Para o Collector, environment e workspace precisam estar no RG e subscription
selecionados, na mesma regiao e ja vinculados. O environment precisa permitir
o ingresso publico exigido pelo Collector. Reveja isolamento e custos antes de
compartilhar essa base entre componentes.

**Pronto quando:** os recursos existem, estao acessiveis pelas identidades
previstas e o armazenamento de estados possui autenticacao, protecao e escopo
definidos. Nao use leitura integral de state como teste: ele pode conter secrets.

Referencias: [base dos runners](https://github.com/PlatformEngineering-Core/gh-runners#readme),
[estados e pre-requisitos da plataforma](https://github.com/PlatformEngineering-Core/platform-enablement#readme)
e [infraestrutura do Collector](https://github.com/PlatformEngineering-Core/OTel-Collector#infrastructure).

## 2. Publicar gh-actions e disponibilizar os templates

Publique a versao aprovada do `gh-actions` na Core e configure o acesso pelos
repositorios consumidores. Prepare o repositorio mirror na organizacao Apps e
execute a sincronizacao autorizada pelo workflow `Sync-Mirror.yml`.
As aplicacoes devem consumir `PlatformEngineering-Apps/gh-actions`; nao a origem
Core privada. Confirme tambem as referencias usadas pelos workflows da plataforma.

Disponibilize os templates que serao oferecidos: APIs .NET, Java, Node e Python,
frontend React e Functions conforme o catalogo publicado. Uma pasta local nao
prova publicacao ou acesso ao repositorio remoto. Confira os callers e os targets
suportados antes de oferecer uma combinacao no portal.

**Pronto quando:** os `uses` resolvem arquivos/refs publicados, o mirror Apps esta
atualizado e a identidade do scaffold consegue ler os templates selecionados.
Templates sao fontes de scaffold, nao servicos que precisam ficar rodando.

Referencia: [gh-actions](https://github.com/PlatformEngineering-Core/gh-actions#readme).

## 3. Subir gh-runners

Configure as credenciais do repositorio conforme o guia dos runners. Publique
primeiro a imagem base e depois as imagens especializadas. O workflow
`build-and-push.yml` usa `ubuntu-latest`, inclusive para criar/atualizar os Jobs:
ele nao depende de um runner `platform` ja registrado. Confira os inputs de
build e deploy antes do dispatch inicial.

Depois execute o caminho documentado de criacao/atualizacao dos Jobs com
`scripts/create-all-jobs.sh`, diretamente ou pelo workflow. A base da etapa 1
precisa existir. As imagens dos runners ficam no GHCR; o ACR das aplicacoes
sera criado posteriormente pela plataforma.

| Pool | Organizacao do registro e do scaler | Label de stack |
| --- | --- | --- |
| Plataforma | Core | `platform` |
| Aplicacoes .NET | Apps | `dotnet` |
| Aplicacoes Java | Apps | `java` |
| Aplicacoes Node e consumidores compativeis | Apps | `node` |
| Aplicacoes Python | Apps | `python` |

Os pools usam tokens de organizacao separados. Confira `GITHUB_URL`,
`RUNNER_LABELS`, owner, `runnerScope`, labels e autenticacao do scaler, alem do
acesso do runner group aos repositorios. Os mesmos labels precisam casar com
`runs-on`. O runner `platform` da Core nao atende jobs de repos Apps.

**Pronto quando:** um job real compativel consegue sair da fila, iniciar uma
execucao ACA e concluir. Scale-to-zero pode deixar o runner ausente/offline em
repouso; isso, sozinho, nao caracteriza falha. A validacao do pool `platform`
pode ser concluida com o fluxo autorizado da etapa 4, e a dos pools Apps com
a aplicacao piloto. Docker CLI instalado no runner nao significa Docker daemon
disponivel; preserve a estrategia de build compativel do projeto.

Referencias: [runners](https://github.com/PlatformEngineering-Core/gh-runners#readme)
e [autenticacao](https://github.com/PlatformEngineering-Core/gh-runners/blob/main/docs/GITHUB-APP-SETUP.md).

## 4. Configurar platform-enablement por company

Na raiz do repositorio `platform-enablement`, siga esta sequencia:

1. `scripts/01-New-TerraformInfrastructureServicePrincipal.ps1`: identidade de
   automacao Azure. Revise as permissoes requeridas e seu escopo antes de aplicar.
2. `scripts/02-New-EnvAndSecretsByCompany.ps1`: Environment da company com
   credenciais Azure, acesso ao storage, Terraform/GitHub e ferramentas de CI.
3. Configure `GH_TARGET_APPS` e confira subscriptions, storage e destinos
   realmente consumidos pelos workflows. Use `scripts/91-Set-PlatformSecrets.ps1`
   para o cadastro central quando aplicavel, conforme seus destinos documentados.
4. Confira a confianca OIDC imutavel. O utilitario
   `scripts/90-Add-ImmutableOidcFic.ps1` atende ajustes desse contrato; nao e uma
   etapa para executar indiscriminadamente em toda identidade.

Nao confunda a identidade Terraform, a identidade do pipeline de cada componente,
a Managed Identity do backend e as identidades das apps. OIDC nao elimina todas
as outras credenciais da plataforma. Protecoes de Environment e aprovacoes
precisam ser configuradas separadamente.

**Pronto quando:** a company esta cadastrada, as configuracoes obrigatorias
existem e os workflows publicados selecionam o Environment e o runner corretos.
Ainda nao e necessario criar manualmente ACR, AKS ou RG de cada futura app.

Referencia: [onboarding de company](https://github.com/PlatformEngineering-Core/platform-enablement/blob/main/docs/ONBOARDING-COMPANY.md).

## 5. Implantar OTel-Collector e cadastrar a telemetria central

Para ter observabilidade desde a primeira app, conclua esta etapa antes do piloto.
O create admite telemetria central ausente, mas isso nao atende a verificacao
completa de observabilidade deste guia.

Na raiz de `OTel-Collector`:

1. Execute `scripts/01-New-OtelCollectorIdentity.ps1` para a company exata,
   selecionando a base existente e o backend de state correto.
2. Execute `scripts/02-Configure-RuntimeTelemetry.ps1`, opcao **1**, para
   configurar Grafana e criar/reutilizar o par bearer. O script tambem cadastra
   o header secreto no Environment correspondente de `platform-enablement`.
3. Execute `Deploy OTel Collector` em `.github/workflows/deploy.yml`, revise
   o plano e aprove o apply pelo procedimento do componente.
4. Cadastre a URL base HTTPS resultante com a opcao **2** do mesmo script.
   Alternativamente, use o `scripts/93-Set-CollectorRuntimeTelemetry.ps1` da
   plataforma para o cadastro central, preservando o bearer ja configurado.
5. Valide envio autenticado e recebimento real no Grafana. Os scripts em
   `scripts/test_remoto/` ajudam a testar logs, metricas e traces.

O API Token Grafana pertence somente a saida Collector -> Grafana; os clientes
usam o bearer Collector. Nao coloque nenhum deles no bundle React. O script
`03-Set-TemplateRuntimeTelemetry.ps1` e **opcional**, para configurar diretamente
templates Core; nao e requisito do create da plataforma.

**Pronto quando:** Collector saudavel, endpoint/header central coerentes e sinais
recebidos no destino. Configuracao local ou HTTP de sucesso nao comprova, sozinho,
recebimento no Grafana. Nao rotacione tokens apenas para repetir o setup.

Referencia: [Collector e rotacao](https://github.com/PlatformEngineering-Core/OTel-Collector#readme).

## 6. Preparar RavenDB e implantar iDPw-backend

Prepare primeiro o RavenDB: URL, database, certificado de cliente com acesso
adequado e conectividade de saida do ACA autorizada pelo servico. Nao presuma
que o IP de ingresso do ACA Environment seja o IP de egresso do backend.

Configure o GitHub App do backend e instale-o na Core para acesso ao repositorio
de workflows e na organizacao Apps para leituras das aplicacoes. Uma nova
organizacao de destino exige sua propria instalacao/autorizacao.

Na raiz de `iDPw-backend`:

1. Execute `scripts/New-OidcIdentity.ps1` e configure os identificadores OIDC.
   Revise/atribua RBAC separadamente: o bootstrap nao faz essas atribuicoes.
2. Use `scripts/Set-RepoSecretsAndVars.ps1` para a topologia, RavenDB, storage,
   GitHub App e acesso a imagem privada no GHCR. Sao settings de repositorio,
   nao o Environment da company usado por `platform-enablement`.
3. Publique pelo `.github/workflows/ci-cd.yml`. O workflow usa GitHub-hosted
   runner; a imagem fica no GHCR. O ACA Environment deve existir antes do deploy.
4. Confirme os dois processos: API e scheduler. Valide RavenDB e leituras do
   GitHub/storage sem depender apenas de uma resposta na raiz HTTP.
5. Com a Managed Identity criada, cadastre seu object ID em
   `IDPW_BACKEND_PRINCIPAL_ID` no Environment da company, antes de provisionar
   recursos cuja leitura pelo portal dependa dela. Revise o uso de
   `platform-enablement/scripts/92-Grant-IdpwBackendReader.ps1` para leitura ARM
   e drift, conforme o escopo aprovado.

O pipeline configura leitura dos blobs para a identidade do backend. Leitura
ARM, acesso de dados a cofres/stores e permissao de dispatch GitHub sao controles
distintos. Nao conceda Contributor amplo para resolver uma consulta pontual.

**Pronto quando:** revisao Ready com a imagem esperada, persistencia funcional,
API/scheduler estaveis, company consultavel e acessos de leitura autorizados.
O vinculo do ACA ao SWA sera feito na proxima etapa.

Referencias: [backend](https://github.com/PlatformEngineering-Core/iDPw-backend#readme)
e [GitHub App](https://github.com/PlatformEngineering-Core/iDPw-backend/blob/main/docs/GITHUB-APP-SETUP.md).

## 7. Implantar iDPw e integrar o acesso

Na raiz de `iDPw`, execute `scripts/New-OidcIdentity.ps1`, configure RBAC do
deploy no escopo previsto e cadastre settings com
`scripts/Set-RepoSecretsAndVars.ps1`. Publique pelo `.github/workflows/ci-cd.yml`,
que usa GitHub-hosted runner e prepara/publica o Static Web App.

Configure os provedores Easy Auth usados: Microsoft com Managed Identity/FIC
e/ou GitHub OAuth App, incluindo URLs de retorno do SWA real. Convide os usuarios
com a role `idp_user`. Vincule o ACA do backend ao SWA como **linked backend**,
verificando suporte regional e topologia antes de contratar os recursos. O
workflow atual cria o SWA com SKU **Standard**, necessario ao custom auth usado.
O proxy local Vite e o usuario simulado de desenvolvimento nao substituem essa
integracao nem a autenticacao em producao.

**Pronto quando:** login autorizado funciona, `/api` chega ao backend vinculado,
company/catalogo carregam e consultas do portal estao autorizadas. Nao basta
o HTML abrir. Nunca exponha secrets em variaveis publicas do frontend.

Referencia: [portal, autenticacao e CI/CD](https://github.com/PlatformEngineering-Core/iDPw#readme).

## 8. Criar uma aplicacao piloto e validar o fluxo completo

Com autorizacao de custo e escopo, escolha uma stack e um target realmente
publicados/suportados. Pelo portal, solicite uma aplicacao piloto na company
configurada e acompanhe seu `control_id` e o workflow `terraform-create.yml`.

O primeiro create provisiona, conforme as escolhas e recursos ja existentes:

- Infra cross da company, como RG e ACR compartilhado.
- Infra da area/ambiente, como RG, observabilidade e ACA Environment ou AKS.
- Recursos da app, identidade, configuracao GitHub, scaffold e runtime settings.
- No caminho AKS, bootstrap do cluster/ArgoCD e wiring GitOps conforme o workflow.

Esses recursos de aplicacao nao devem ser precriados manualmente para tentar
substituir os modulos Terraform. No AKS, deploy posterior atualiza a tag no Git
e o ArgoCD reconcilia; confirme capacidade, probes e Sync/Health. Outros targets
possuem contratos proprios: um piloto ACA nao certifica AKS, Functions ou SWA.

Checklist de aceite:

- [ ] Runner da Core executa o provisionamento e libera o lock corretamente.
- [ ] Repo Apps tem branches, Environments, callers e scaffold esperados.
- [ ] CI/CD usa o mirror Apps, encontra seu runner e passa pelos gates exigidos.
- [ ] Deploy confirma a imagem/artefato esperado e a app responde no acesso previsto.
- [ ] States foram persistidos e o backend apresenta a app no portfolio.
- [ ] Logs, metricas e traces instrumentados chegam ao Grafana com o servico correto.
- [ ] Promote para outro ambiente foi verificado, caso faca parte do aceite contratado.

Instrumentacao e injecao de settings variam por target/template. No frontend,
telemetria autenticada exige um relay server-side separado; nao distribua bearer
ao navegador. Cadastro de secrets no GitHub nao comprova instrumentacao de runtime.

Nao use destroy como verificacao automatica do setup. Ao encerrar o piloto,
avalie os recursos compartilhados e estados afetados antes de autorizar limpeza.
Cada nova company repete seu onboarding, configuracao de Collector e autorizacoes;
nao precisa de uma copia inteira dos componentes centrais se o desenho permite reuso.

## Limites e custos

Esta ordem foi documentada a partir dos arquivos locais em novembro de 2026;
confira os commits e contratos publicados antes de executar. A documentacao nao
executa deploys, nao configura contas externas e nao confirma permissoes reais.

AKS, replicas ACA, storage, logs, RavenDB, Grafana e requisitos de SKU do portal
podem gerar custos. Valide regiao, quota, rede e retencao antes do provisionamento.
Nao remova locks, nao descarte state e nao desabilite gates de seguranca para
contornar falhas de bootstrap. Registre separadamente o que foi configurado,
publicado, implantado e confirmado no ambiente real.
