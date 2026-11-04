---
name: create-repository
description: Cria um novo repositório GitHub na org ebravo-br do zero seguindo todos os padrões da org — branch protection, permissões de time, workflows CI/CD (ci/deploy/rollback/project-automation/issue-epic), estrutura infra/, pipeline.yml e vínculo ao épico correto no project "Ebravo Projetos". Use quando pedirem para "criar um repositório", "criar repo novo", "novo serviço na ebravo-br", ou via /create-repository <repo> <epic> [tecnologia] [componente].
---

# Create Repository (ebravo-br)

Cria um novo repositório GitHub na org `ebravo-br` do zero seguindo todos os
padrões da org: branch protection, permissões de time, workflows CI/CD,
estrutura `infra/`, `pipeline.yml` e vínculo ao épico correto no projeto
"Ebravo Projetos".

## Input: $ARGUMENTS

Formato esperado: `<repo> <epic> [tecnologia] [componente]`

Exemplos:
- `/create-repository ebdocs-novo-servico EBDOCS spring-boot-java`
- `/create-repository acg-portal ACG nodejs`
- `/create-repository meu-servico DATAPAT`

Parâmetros:
- **repo**: nome do repositório (sem org prefix). Será criado em `ebravo-br/`.
- **epic**: épico obrigatório (ver lista abaixo). **DEVE ser fornecido.**
- **tecnologia**: `spring-boot-java`, `java`, `nodejs`, `angular`, `python` ou `docker`. Se ausente, perguntar antes de continuar.
  `docker` é para imagem montada só pelo Dockerfile, sem etapa de build de linguagem (ex.: empacotar um software de terceiros como o GLPI). A esteira não roda build e lê a versão do `package.json` da raiz (`cd.yml`, fallback de `version_file`).
- **componente**: nome do serviço/componente. Se ausente, usar o nome do repo.

---

## Épicos disponíveis

| Épico | Prefixo de repo que mapeia automaticamente |
|---|---|
| `TRAMAR` | `tramar-*` ou org `tramar-br` |
| `EBDOCS` | `ebdocs-*` |
| `EBCARE` | `ebcare-*` |
| `CEMITERIO` | `cemiterio*` |
| `ERS` | `ers-*` |
| `ATOS` | `*atos*` (contém "atos") |
| `DATAPAT` | `datapat-*` |
| `TOTEM` | `totem-*` |
| `ACG` | `acg-*` |
| `SENSIVE` | `sensive-*` |
| `EBRAG` | `rag-*` |
| `INFRA` | `ebravo-agents*` |
| `HELPDESK` | `ebravo-helpdesk*` |

Se o épico informado não estiver nessa lista, **pare** e informe o usuário que é necessário:
1. Adicionar a opção ao campo "Épico" no GitHub Project #2 (`ebravo-br / Ebravo Projetos`)
2. Adicionar a regra de mapeamento em `issue-epic.yml` no `cicd-templates`

---

## Instruções

### Passo 0 — Parsear argumentos e validar épico (BLOQUEANTE)

A partir de `$ARGUMENTS`:
- **repo**: primeiro argumento. Remover prefixo `ebravo-br/` se presente; re-adicionar internamente.
- **epic**: segundo argumento. Normalizar para maiúsculas.
- **tecnologia**: terceiro argumento. Se ausente, **pare e pergunte** antes de continuar.
- **componente**: quarto argumento. Se ausente, usar o nome do repo (sem org prefix).

**Validação do épico — OBRIGATÓRIA:**

Se `epic` não foi informado ou está vazio, **pare** e exiba:

```
Épico é obrigatório. Qual épico deseja associar ao repositório?

Épicos disponíveis:
  TRAMAR     → repos tramar-* ou org tramar-br
  EBDOCS     → repos ebdocs-*
  EBCARE     → repos ebcare-*
  CEMITERIO  → repos cemiterio*
  ERS        → repos ers-*
  ATOS       → repos *atos*
  DATAPAT    → repos datapat-*
  TOTEM      → repos totem-*
  ACG        → repos acg-*
  SENSIVE    → repos sensive-*
  EBRAG      → repos rag-*
  INFRA      → repos ebravo-agents*
  HELPDESK   → repos ebravo-helpdesk*

Se o épico do projeto ainda não existe nessa lista, informe o nome desejado
e explique que precisará ser criado manualmente antes de continuar.
```

Se `epic` foi informado mas **não está na lista acima**, **pare** e exiba:

```
⚠ Épico "<X>" não reconhecido.

Para criar um novo épico é necessário:
1. Acessar o GitHub Project #2 (ebravo-br / Ebravo Projetos)
   → Aba "Settings" → "Custom fields" → campo "Épico" → adicionar opção "<X>"
2. Abrir PR em ebravo-br/cicd-templates adicionando a regra abaixo em
   .github/workflows/issue-epic.yml (na sequência de elif, antes do `else`):

   elif [[ "$REPO_LC" == <prefixo>* ]]; then echo "epic=<X>" >> "$GITHUB_OUTPUT"

Após criar o épico no projeto e mesclar o PR, criar uma tag nova do cicd-templates
no commit mesclado (ver "Tag dos stubs" no Passo 5) e executar este comando novamente.

Épicos válidos agora: TRAMAR, EBDOCS, EBCARE, CEMITERIO, ERS, ATOS, DATAPAT, TOTEM, ACG, SENSIVE, EBRAG, INFRA, HELPDESK
```

Não avançar para o Passo 1 sem épico válido confirmado.

---

### Passo 1 — Verificar que o repositório ainda NÃO existe

```bash
gh repo view ebravo-br/<repo> 2>/dev/null
```

Se o repositório já existir, **abortar** e informar:

```
⚠ O repositório ebravo-br/<repo> já existe.
Para configurar o pipeline nele, use: /setup-pipeline <repo> <tecnologia>
```

---

### Passo 2 — Criar o repositório

```bash
gh repo create ebravo-br/<repo> \
  --private \
  --description "<componente> — <epic>" \
  --add-readme
```

`--add-readme` cria um commit inicial na branch `main`, necessário para que a branch protection seja aplicada na sequência.

Após a criação, aplicar as configurações padrão da org:

```bash
gh api -X PATCH repos/ebravo-br/<repo> \
  --field delete_branch_on_merge=true \
  --field allow_squash_merge=false \
  --field allow_merge_commit=true \
  --field allow_rebase_merge=false
```

Exibir a URL do repositório criado.

---

### Passo 3 — Configurar permissões de times

#### 3a — Listar times da org

```bash
gh api orgs/ebravo-br/teams --jq '[.[] | {slug, name}]'
```

Exibir os times encontrados ao usuário.

#### 3b — Aplicar permissões

Regra padrão:
- Times cujo slug contém `admin` ou `ops` → permissão `admin`
- Demais times → permissão `push`

Se a classificação automática for ambígua (ex: nenhum time claramente admin, ou múltiplos times admin), **perguntar ao usuário** qual nível dar a cada time antes de prosseguir.

```bash
# Para cada time:
gh api -X PUT orgs/ebravo-br/teams/<slug>/repos/ebravo-br/<repo> \
  --field permission=<push|admin>
```

---

### Passo 4 — Configurar Branch Protection na `main`

```bash
gh api -X PUT repos/ebravo-br/<repo>/branches/main/protection \
  --input - <<'EOF'
{
  "required_status_checks": {
    "strict": false,
    "contexts": []
  },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "restrictions": {
    "users": ["juniorebravo"],
    "teams": [],
    "apps": ["github-actions"]
  },
  "allow_force_pushes": false,
  "allow_deletions": false
}
EOF
```

---

### Passo 5 — Criar stubs de workflow CI/CD

O repo foi recém-criado — não há SHA preexistente. Usar PUT direto para todos os arquivos.

**Tag dos stubs: `@v5`.** Todos os stubs apontam para a mesma tag. Nunca mover a `v1` (compartilhada com a `tramar-br`, V8 no `ebravo-infra`) nem mover tag existente para incluir mudança nova: criar a próxima (`v6`...) e atualizar este passo. Uma regra nova no `issue-epic.yml` só vale para os repos cujo stub aponta para uma tag que a contém.

Para repo que roda **só em homologação**, travar `target_env: [homologacao]` e `deploy_platform: [ec2]` nos stubs de deploy e rollback (como no `ebscan-app`).

#### 5a — `.github/workflows/ci.yml`

```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop, 'feature/**', 'bug/**', 'task/**']

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    if: toJson(github.event.commits) != '[]'
    uses: ebravo-br/cicd-templates/.github/workflows/ci.yml@v5
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/workflows/ci.yml \
  --field message="ci: instalar stub de CI ebravo-br" \
  --field content="$ENCODED"
```

#### 5b — `.github/workflows/deploy.yml`

```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/cd.yml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      target_env:
        description: 'Ambiente alvo'
        type: choice
        required: true
        default: homologacao
        options: [homologacao, producao]
      deploy_platform:
        description: 'Plataforma de deploy'
        type: choice
        required: true
        default: ec2
        options: [ec2, ecs, gke]

permissions:
  contents: write
  id-token: write
  pull-requests: write

concurrency:
  group: deploy-${{ github.ref }}-${{ inputs.target_env }}
  cancel-in-progress: false

jobs:
  cd:
    uses: ebravo-br/cicd-templates/.github/workflows/cd.yml@v5
    secrets: inherit
    with:
      target_env:      ${{ inputs.target_env }}
      deploy_platform: ${{ inputs.deploy_platform }}
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/workflows/deploy.yml \
  --field message="ci: instalar stub de deploy ebravo-br" \
  --field content="$ENCODED"
```

#### 5c — `.github/workflows/rollback.yml`

```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/rollback.yml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Tag da imagem para voltar (ex: 1.5.0). Obrigatória.'
        type: string
        required: true
      target_env:
        description: 'Ambiente alvo'
        type: choice
        required: true
        default: homologacao
        options: [homologacao, producao]
      deploy_platform:
        description: 'Plataforma de deploy'
        type: choice
        required: true
        default: ec2
        options: [ec2, ecs, gke]

permissions:
  contents: read
  id-token: write

concurrency:
  group: rollback-${{ github.ref }}-${{ inputs.target_env }}
  cancel-in-progress: false

jobs:
  rollback:
    uses: ebravo-br/cicd-templates/.github/workflows/rollback.yml@v5
    secrets: inherit
    with:
      image_tag:       ${{ inputs.image_tag }}
      target_env:      ${{ inputs.target_env }}
      deploy_platform: ${{ inputs.deploy_platform }}
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/workflows/rollback.yml \
  --field message="ci: instalar stub de rollback ebravo-br" \
  --field content="$ENCODED"
```

#### 5d — `.github/workflows/project-automation.yml`

```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/project-automation.yml
name: Project Automation

on:
  pull_request:
    types: [opened, closed]

permissions:
  contents: write

jobs:
  project:
    uses: ebravo-br/cicd-templates/.github/workflows/project-automation.yml@v5
    secrets: inherit
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/workflows/project-automation.yml \
  --field message="ci: instalar stub de project automation ebravo-br" \
  --field content="$ENCODED"
```

#### 5e — `.github/workflows/issue-epic.yml`

```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/issue-epic.yml
name: Issue Epic Assignment

on:
  issues:
    types: [opened]

permissions:
  contents: read

jobs:
  epic:
    uses: ebravo-br/cicd-templates/.github/workflows/issue-epic.yml@v5
    secrets: inherit
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/workflows/issue-epic.yml \
  --field message="ci: instalar stub de issue epic ebravo-br" \
  --field content="$ENCODED"
```

#### 5f — `.github/workflows/branch-naming.yml`

```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/branch-naming.yml
name: Branch Naming
on:
  create:
permissions:
  contents: write
jobs:
  rename:
    uses: ebravo-br/cicd-templates/.github/workflows/branch-naming.yml@v5
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/workflows/branch-naming.yml \
  --field message="ci: instalar stub de branch naming ebravo-br" \
  --field content="$ENCODED"
```

---

### Passo 6 — Criar `.github/pipeline.yml`

Montar o conteúdo de acordo com a tecnologia:

Para `spring-boot-java` / `java`:
```yaml
componente: <componente>
tecnologia: <tecnologia>
jdk: jdk17   # omitir se for o padrão 17
```

Para `nodejs` / `angular`:
```yaml
componente: <componente>
tecnologia: <tecnologia>
node: '20'   # omitir se for o padrão 20
```

Para `python`:
```yaml
componente: <componente>
tecnologia: <tecnologia>
python: '3.12'   # omitir se for o padrão 3.12
```

Para `docker`:
```yaml
componente: <componente>
tecnologia: docker
```

E criar também o `package.json` da raiz, que só carrega a versão (a esteira lê e grava `version` nele):
```json
{
  "name": "<repo>",
  "version": "1.0.0",
  "private": true
}
```

> O nome do componente define onde o `cd.yml` procura o contrato: componente single-repo que não é Java nem Python é tratado como `web`, e o contrato fica em `infra/deploy/web/infra.yaml`. O validador do `ebravo-infra` exige que componente em `deploy/web/` termine em `-web`. Para tecnologia `docker` com deploy em ECS, **usar componente terminando em `-web`**.

Para monorepo (api + web):
```yaml
monorepo: true
api:
  componente: <componente>-api
  tecnologia: spring-boot-java
  jdk: jdk17
web:
  componente: <componente>-web
  tecnologia: nodejs
  node: '20'
```

Criar via API (codificar em base64):
```bash
ENCODED=$(printf '%s' "$CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/.github/pipeline.yml \
  --field message="ci: adicionar pipeline.yml" \
  --field content="$ENCODED"
```

---

### Passo 7 — Criar estrutura `infra/`

O repo é novo — sempre criar a partir do template da tecnologia.

#### 7a — `infra/docker/Dockerfile`

Para `spring-boot-java` / `java`:
```dockerfile
FROM bellsoft/liberica-openjdk-alpine-musl:21

RUN apk add --no-cache tzdata \
 && cp /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime \
 && echo "America/Sao_Paulo" > /etc/timezone \
 && apk del tzdata

ENV TZ=America/Sao_Paulo
ENV LANG=pt_BR.UTF-8
ENV LANGUAGE=pt_BR.UTF-8
ENV LC_ALL=pt_BR.UTF-8
ENV JAVA_OPTS="-XX:MaxRAMPercentage=70 -XX:MinRAMPercentage=70 -XX:InitialRAMPercentage=50 -DBPL_JAVA_NMT_ENABLED=false"

COPY ./target/<componente>-*.jar /deployment/<componente>.jar

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /deployment/<componente>.jar"]
```

Para `nodejs` / `angular`:
```dockerfile
FROM nginx:alpine

RUN apk add --no-cache tzdata \
 && cp /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime \
 && echo "America/Sao_Paulo" > /etc/timezone \
 && apk del tzdata

ENV TZ=America/Sao_Paulo

COPY ./config/nginx.conf /etc/nginx/conf.d/default.conf
COPY /dist/<componente> /usr/share/nginx/html

EXPOSE 80
```

Para `python`:
```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends tzdata \
 && cp /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime \
 && echo "America/Sao_Paulo" > /etc/timezone \
 && rm -rf /var/lib/apt/lists/*

ENV TZ=America/Sao_Paulo

WORKDIR /app
COPY dist/ .

ENTRYPOINT ["python", "-m", "<componente>"]
```

Para `docker`: não há template. O Dockerfile é o do software empacotado (`FROM` da imagem oficial fixada por versão e digest, mais a configuração própria). Criar um placeholder que falhe o build até ser substituído:
```dockerfile
# Substituir pelo Dockerfile real do componente.
FROM scratch
```

> **Monorepo**: criar `infra/docker/api/Dockerfile` (Java) e `infra/docker/web/Dockerfile` (Node/Angular).

Criar via API:
```bash
ENCODED=$(printf '%s' "$DOCKERFILE" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/infra/docker/Dockerfile \
  --field message="infra: adicionar Dockerfile template" \
  --field content="$ENCODED"
```

#### 7b — `infra/compose/docker-compose.yml`

```yaml
version: '3.8'
services:
  <componente>:
    image: <componente>:latest
    ports:
      - "8080:8080"
```

```bash
ENCODED=$(printf '%s' "$DC_CONTENT" | base64)
gh api -X PUT repos/ebravo-br/<repo>/contents/infra/compose/docker-compose.yml \
  --field message="infra: adicionar docker-compose.yml stub" \
  --field content="$ENCODED"
```

---

### Passo 8 — Verificar mapeamento de épico em `issue-epic.yml`

Aplicar a mesma lógica de mapeamento do `issue-epic.yml` ao nome do repo (em lowercase) e comparar com o épico informado pelo usuário:

```
REPO_LC = lowercase(repo_name)

tramar-br org            → TRAMAR
ebdocs*                  → EBDOCS
ebcare*                  → EBCARE
tramar*                  → TRAMAR
cemiterio*               → CEMITERIO
ers*                     → ERS
*atos*                   → ATOS
datapat*                 → DATAPAT
totem*                   → TOTEM
acg*                     → ACG
sensive*                 → SENSIVE
rag*                     → EBRAG
ebravo-agents*           → INFRA
ebravo-helpdesk*         → HELPDESK
(sem match)              → (nenhum)
```

| Cenário | Ação |
|---|---|
| `auto == epic informado` | Confirmar: "✓ O prefixo do repo mapeia automaticamente para `<EPIC>`." |
| `auto == (nenhum)` e usuário informou epic | ⚠ Aviso: o prefixo não mapeia automaticamente. Exibir linha a adicionar em `issue-epic.yml`. |
| `auto != epic informado` (mapeamento diferente) | ⚠ Aviso: conflito de mapeamento. Exibir linha a corrigir em `issue-epic.yml`. |

Quando mudança em `issue-epic.yml` for necessária, exibir o diff exato:

```
Para que issues do repo ebravo-br/<repo> sejam tagueadas como <EPIC>,
adicione a linha abaixo em .github/workflows/issue-epic.yml
(dentro do step "Detectar épico", antes do `else` final):

  elif [[ "$REPO_LC" == <prefixo>* ]]; then echo "epic=<EPIC>" >> "$GITHUB_OUTPUT"

Esta alteração requer um PR em ebravo-br/cicd-templates.
```

---

### Passo 9 — Verificar secrets e variáveis da org

```bash
gh api orgs/ebravo-br/actions/secrets | jq '[.secrets[].name]'
gh api orgs/ebravo-br/actions/variables/DEPLOY_PROD_ALLOWED 2>/dev/null
```

Secrets obrigatórias na org: `APP_AUTOMACAO_PRIVATE_KEY`, `SSH_KEY_HOMOL`, `SSH_KEY_PROD`. Variáveis da org: `APP_AUTOMACAO_ID`, `DEPLOY_PROD_ALLOWED`, `ECS_CLUSTER_HOMOL`, `ECS_CLUSTER_PROD`.

O bump de versão usa o GitHub App (`APP_AUTOMACAO_ID` + `APP_AUTOMACAO_PRIVATE_KEY`); `GH_TOKEN_BUMP` é só fallback e pode não existir.

`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` são o caminho antigo. Repo novo usa **OIDC**: criar os environments `homologacao` e `producao` no repo, cada um com a variável `AWS_DEPLOY_ROLE_ARN`:

```bash
for env in homologacao producao; do
  gh api -X PUT repos/ebravo-br/<repo>/environments/$env --silent
done
gh variable set AWS_DEPLOY_ROLE_ARN -R ebravo-br/<repo> --env homologacao \
  --body arn:aws:iam::851479049298:role/github-actions-deploy-homol
gh variable set AWS_DEPLOY_ROLE_ARN -R ebravo-br/<repo> --env producao \
  --body arn:aws:iam::851479049298:role/github-actions-deploy-prod
```

A role só aceita o repo depois que ele entrar na trust policy, no `ebravo-infra` (PR pela skill `mudanca-terraform`): `deploy_repositories` em `terraform/environments/prod/locals.tf`, o ID numérico em `repository_ids` (`prod/main.tf`) e a lista de homologação em `homol/main.tf`. O ID sai de `gh api repos/ebravo-br/<repo> -q .id`. Listar isso nos próximos passos do resumo.

Se `PROJECT_PAT` estiver ausente, instruir:
1. `github.com/settings/tokens` → Generate new token (classic), scopes: `project`, `repo`, `read:org`
2. `gh secret set PROJECT_PAT --org ebravo-br --visibility all --body "<TOKEN>"`
3. `gh secret set PROJECT_PAT --org tramar-br --visibility all --body "<TOKEN>"` (mesmo value)

---

### Passo 10 — Resumo final

```
Repositório
ebravo-br/<repo>                               ✓ criado  (https://github.com/ebravo-br/<repo>)

Épico
Épico informado:                               <EPIC>
Mapeamento automático (issue-epic.yml):        ✓ correto | ⚠ sem mapeamento — ver instruções acima

Times e permissões
<team-slug> (push)                             ✓ configurado
<admin-team-slug> (admin)                      ✓ configurado

Workflows
.github/workflows/ci.yml                       ✓ criado
.github/workflows/deploy.yml                   ✓ criado
.github/workflows/rollback.yml                 ✓ criado
.github/workflows/project-automation.yml       ✓ criado
.github/workflows/issue-epic.yml               ✓ criado
.github/workflows/branch-naming.yml            ✓ criado
.github/pipeline.yml                           ✓ criado

Infra
infra/docker/Dockerfile                        ✓ criado  (template <tecnologia>)
infra/compose/docker-compose.yml               ✓ criado

GitHub
Branch protection (main)                       ✓ configurada  (1 reviewer, enforce_admins OFF, push: juniorebravo)
Org secrets (AWS + SSH)                        ✓ ok | ⚠ faltam: [lista]
Environments homologacao/producao (OIDC)      ✓ criados com AWS_DEPLOY_ROLE_ARN
Trust OIDC no ebravo-infra                     ⚠ pendente — PR em ebravo-infra
Org secret PROJECT_PAT                         ✓ ok | ⚠ ausente — automação de projeto não vai funcionar
Org variable DEPLOY_PROD_ALLOWED               ✓ ok | ⚠ ausente

Próximos passos:
- [listar apenas o que ficou pendente]
- Clone: git clone git@github.com:ebravo-br/<repo>.git
```
