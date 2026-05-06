# Setup Pipeline ebravo-br

Configura o pipeline CI/CD completo da ebravo-br em um repositório GitHub:
cria os stubs de workflow, o `pipeline.yml`, proteção de branch e verifica
secrets/variáveis da org.

## Input: $ARGUMENTS

Formato esperado: `<repo> [tecnologia] [componente]`

Exemplos:
- `/setup-pipeline meu-servico spring-boot-java`
- `/setup-pipeline ebravo-br/meu-servico spring-boot-java meu-servico`
- `/setup-pipeline meu-front nodejs`

---

## Instruções

### Passo 0 — Parsear argumentos

A partir de `$ARGUMENTS`:
- **repo**: nome do repo ou `org/repo`. Se não tiver `/`, prefixar com `ebravo-br/`.
- **tecnologia**: `spring-boot-java`, `java`, `nodejs`, `angular` ou `python`. Se ausente, perguntar ao usuário antes de continuar.
- **componente**: nome do serviço/componente. Se ausente, usar o nome do repo (sem o org prefix).
- **versao extra** (opcional): perguntar apenas se a tecnologia exigir versão não-padrão:
  - Java/Spring → `jdk` (padrão `17`)
  - Node/Angular → `node` (padrão `20`)
  - Python → `python` (padrão `3.12`)

Se `tecnologia` não foi informada, **pare e pergunte** antes de seguir.

---

### Passo 1 — Verificar se o repo existe na org

```bash
gh repo view <org/repo>
```

Se não existir, avisar o usuário e interromper.

---

### Passo 2 — Criar `.github/workflows/ci.yml`

Usar `gh api` para criar ou atualizar o arquivo diretamente no repo. Se o arquivo já existir, obter o SHA atual antes de sobrescrever.

Verificar se o arquivo já existe:
```bash
gh api repos/<org/repo>/contents/.github/workflows/ci.yml 2>/dev/null
```

Conteúdo do arquivo (sempre o mesmo, independente do projeto):

```yaml
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/ci.yml
name: CI

on:
  push:
    branches:
      - main
      - 'feature/**'
      - 'hotfix/**'
      - 'bugfix/**'

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    if: toJson(github.event.commits) != '[]'
    uses: ebravo-br/cicd-templates/.github/workflows/ci.yml@v1
```

Criar via API (codificar em base64):
```bash
CONTENT=$(cat <<'EOF'
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/ci.yml
name: CI

on:
  push:
    branches:
      - main
      - 'feature/**'
      - 'hotfix/**'
      - 'bugfix/**'

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    if: toJson(github.event.commits) != '[]'
    uses: ebravo-br/cicd-templates/.github/workflows/ci.yml@v1
EOF
)
ENCODED=$(printf '%s' "$CONTENT" | base64)
# Se arquivo já existir, incluir "sha": "<sha>" no JSON
gh api -X PUT repos/<org/repo>/contents/.github/workflows/ci.yml \
  --field message="ci: instalar stub de CI ebravo-br" \
  --field content="$ENCODED"
```

---

### Passo 3 — Criar `.github/workflows/deploy.yml`

Mesmo processo (verificar SHA se existir).

Conteúdo:

```yaml
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
    uses: ebravo-br/cicd-templates/.github/workflows/cd.yml@v1
    secrets: inherit
    with:
      target_env:      ${{ inputs.target_env }}
      deploy_platform: ${{ inputs.deploy_platform }}
```

---

### Passo 3b — Criar `.github/workflows/rollback.yml`

Mesmo processo (verificar SHA se existir).

Conteúdo:

```yaml
# >>> NAO EDITE ESTE ARQUIVO <<<
# Logica em: ebravo-br/cicd-templates/.github/workflows/rollback.yml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Tag da imagem (ex: 1.5.0). Deixe vazio para listar as 10 mais recentes.'
        type: string
        required: false
        default: ''
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
    uses: ebravo-br/cicd-templates/.github/workflows/rollback.yml@v1
    secrets: inherit
    with:
      image_tag:       ${{ inputs.image_tag }}
      target_env:      ${{ inputs.target_env }}
      deploy_platform: ${{ inputs.deploy_platform }}
```

---

### Passo 3c — Criar `.github/workflows/project-automation.yml`

Mesmo processo (verificar SHA se existir).

Conteúdo:

```yaml
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
    uses: ebravo-br/cicd-templates/.github/workflows/project-automation.yml@v1
    secrets: inherit
```

---

### Passo 3d — Criar `.github/workflows/issue-epic.yml`

Mesmo processo (verificar SHA se existir).

Conteúdo:

```yaml
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
    uses: ebravo-br/cicd-templates/.github/workflows/issue-epic.yml@v1
    secrets: inherit
```

---

### Passo 4 — Criar `.github/pipeline.yml`

Se o arquivo já existir, **não sobrescrever** — apenas avisar o usuário com o conteúdo atual e sugerir revisão.

Montar o conteúdo dinamicamente com base nos argumentos coletados:

Para tecnologias Java/Spring:
```yaml
componente: <componente>
tecnologia: <tecnologia>
jdk: jdk<versao>   # omitir se for o padrão 17
```

Para Node/Angular:
```yaml
componente: <componente>
tecnologia: <tecnologia>
node: <versao>   # omitir se for o padrão 20
```

Para Python:
```yaml
componente: <componente>
tecnologia: <tecnologia>
python: '<versao>'   # omitir se for o padrão 3.12
```

Para monorepo (api + web), usar o formato expandido:
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

---

### Passo 5 — Organizar estrutura `infra/`

Este passo garante que o repo siga a convenção de diretórios da org e que o `cd.yml` encontre o Dockerfile.

#### 5a — Localizar e mover o Dockerfile

Verificar em ordem de prioridade:
```bash
# Já na posição correta?
gh api repos/<org/repo>/contents/infra/docker/Dockerfile 2>/dev/null | jq -r '.sha // empty'

# Na raiz?
gh api repos/<org/repo>/contents/Dockerfile 2>/dev/null | jq -r '.sha // empty'

# Path legado single-repo?
gh api repos/<org/repo>/contents/infra/docker/api/Dockerfile 2>/dev/null | jq -r '.sha // empty'
gh api repos/<org/repo>/contents/infra/docker/web/Dockerfile 2>/dev/null | jq -r '.sha // empty'
```

**Cenários e ações:**

| Situação | Ação |
|---|---|
| `infra/docker/Dockerfile` já existe | Nada a fazer ✓ |
| Dockerfile na raiz | Mover para `infra/docker/Dockerfile`, deletar da raiz |
| `infra/docker/api/Dockerfile` ou `infra/docker/web/Dockerfile` | Mover para `infra/docker/Dockerfile`, deletar o legado |
| Nenhum encontrado | Criar `infra/docker/Dockerfile` a partir do template abaixo |

**Mover Dockerfile (raiz ou path legado → `infra/docker/Dockerfile`):**
```bash
OLD_PATH="Dockerfile"  # ou infra/docker/api/Dockerfile, etc.
OLD_SHA=$(gh api repos/<org/repo>/contents/$OLD_PATH | jq -r '.sha')
CONTENT=$(gh api repos/<org/repo>/contents/$OLD_PATH | jq -r '.content' | tr -d '\n')

gh api -X PUT repos/<org/repo>/contents/infra/docker/Dockerfile \
  --field message="infra: mover Dockerfile para infra/docker/Dockerfile" \
  --field content="$CONTENT"

gh api -X DELETE repos/<org/repo>/contents/$OLD_PATH \
  --field message="infra: remover Dockerfile do path antigo" \
  --field sha="$OLD_SHA"
```

**Criar Dockerfile do zero** (quando não existe), baseado na tecnologia:

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

> **Monorepo**: criar `infra/docker/api/Dockerfile` (Java) e `infra/docker/web/Dockerfile` (Node/Angular) — mesmo padrão acima para cada um.

---

#### 5b — Mover `docker-compose.yml` da raiz para `infra/compose/`

Verificar se existe na raiz:
```bash
gh api repos/<org/repo>/contents/docker-compose.yml 2>/dev/null | jq -r '.sha // empty'
```

Se existir, mover:
```bash
DC_SHA=$(gh api repos/<org/repo>/contents/docker-compose.yml | jq -r '.sha')
DC_CONTENT=$(gh api repos/<org/repo>/contents/docker-compose.yml | jq -r '.content' | tr -d '\n')

gh api -X PUT repos/<org/repo>/contents/infra/compose/docker-compose.yml \
  --field message="infra: mover docker-compose.yml para infra/compose/" \
  --field content="$DC_CONTENT"

gh api -X DELETE repos/<org/repo>/contents/docker-compose.yml \
  --field message="infra: remover docker-compose.yml da raiz" \
  --field sha="$DC_SHA"
```


---

### Passo 6 — Configurar Branch Protection na `main`

Verificar se proteção já existe:
```bash
gh api repos/<org/repo>/branches/main/protection 2>/dev/null
```

Aplicar (ou sobrescrever) com as regras padrão da org:
```bash
gh api -X PUT repos/<org/repo>/branches/main/protection \
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

### Passo 7 — Verificar secrets e variáveis da org

Checar se as secrets obrigatórias existem na org:
```bash
gh api orgs/ebravo-br/actions/secrets | jq '[.secrets[].name]'
```

Secrets esperadas: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `SSH_KEY_HOMOL`, `SSH_KEY_PROD`, `GH_TOKEN_BUMP`, `PROJECT_PAT`

Checar variável de controle de acesso:
```bash
gh api orgs/ebravo-br/actions/variables/DEPLOY_PROD_ALLOWED 2>/dev/null
```

Se alguma estiver faltando, listar claramente o que falta e orientar o usuário a configurar antes do primeiro deploy.

**Secret `GH_TOKEN_BUMP` — obrigatória para deploy funcionar corretamente**

Sem ela, o bump de versão no `pom.xml` / `package.json` é silenciosamente rejeitado pela branch protection e o deploy continua com a versão errada no código.

Se estiver ausente, instruir o usuário a:

1. Acessar `github.com/settings/tokens` → **Generate new token (classic)**
   - Note: `GH_TOKEN_BUMP`
   - Scopes: apenas **`repo`**

2. Cadastrar na org:
```bash
gh secret set GH_TOKEN_BUMP --org ebravo-br --visibility all --body "<TOKEN>"
```

**Secret `PROJECT_PAT` — obrigatória para automação do GitHub Projects funcionar**

O `GITHUB_TOKEN` nativo não tem permissão de escrita em Projects v2. A automação de Kanban (mover issues, atribuir épicos, centralizar issues cross-org no projeto "Ebravo Projetos") usa um PAT clássico scoped no usuário (membro de `ebravo-br` E `tramar-br`).

> **Por que não GitHub App?** Tokens de App são single-installation: um token vê só uma org. A chamada `addProjectV2ItemById` cross-org precisa de **um único** token que enxergue tanto a issue (na org caller) quanto o project (em outra org) — só PAT do usuário consegue.

Se a secret estiver ausente, instruir o usuário a:

1. Acessar `github.com/settings/tokens` → **Generate new token (classic)**
   - Note: `PROJECT_PAT`
   - Scopes: `project`, `repo`, `read:org`
2. Cadastrar na org:
```bash
gh secret set PROJECT_PAT --org ebravo-br --visibility all --body "<TOKEN>"
```

> Para repos da `tramar-br` que reutilizam estes workflows, a **mesma** secret precisa estar cadastrada na org `tramar-br` também (ver `setup-pipeline-tramar`).

---

### Passo 8 — Resumo final

Exibir uma tabela com o status de cada item:

```
Repo:     ebravo-br/<repo>

Workflows
.github/workflows/ci.yml                       ✓ criado | ✓ atualizado
.github/workflows/deploy.yml                   ✓ criado | ✓ atualizado
.github/workflows/rollback.yml                 ✓ criado | ✓ atualizado
.github/workflows/project-automation.yml       ✓ criado | ✓ atualizado
.github/workflows/issue-epic.yml               ✓ criado | ✓ atualizado
.github/pipeline.yml                           ✓ criado | ⚠ ja existia (nao sobrescrito)

Infra
infra/docker/Dockerfile                        ✓ criado | ✓ movido de <origem> | ⚠ ausente
infra/compose/docker-compose.yml               ✓ movido da raiz | — nao havia
GitHub
Branch protection (main)                       ✓ configurada
Org secrets (AWS + SSH)                        ✓ ok | ⚠ faltam: [lista]
Org secret GH_TOKEN_BUMP                       ✓ ok | ⚠ ausente — bump de versao nao vai funcionar
Org secret PROJECT_PAT                         ✓ ok | ⚠ ausente — automacao de projeto nao vai funcionar
Org variable DEPLOY_PROD_ALLOWED               ✓ ok | ⚠ ausente

Proximos passos:
- [listar apenas o que ficou pendente]
```

---

## Pré-requisitos cross-org (uma vez só)

Estas etapas são **manuais** e fora do escopo do `/setup-pipeline`. Necessárias para que a automação do projeto "Ebravo Projetos" funcione com repos das duas orgs.

### 1. Criar o `PROJECT_PAT`

1. Usuário membro de **ebravo-br** E **tramar-br** acessa `github.com/settings/tokens` → **Generate new token (classic)**
   - Note: `PROJECT_PAT`
   - Scopes: `project`, `repo`, `read:org`
   - Expiração: **No expiration** (ou rotacionar manualmente)

2. Cadastrar a **mesma value** em ambas as orgs:

```bash
gh secret set PROJECT_PAT --org ebravo-br --visibility all --body "<TOKEN>"
gh secret set PROJECT_PAT --org tramar-br --visibility all --body "<TOKEN>"
```

### 2. Habilitar acesso cross-org ao `cicd-templates`

Para que repos da `tramar-br` consigam invocar `uses: ebravo-br/cicd-templates/...@v1`:

- **Opção A (recomendada)**: tornar `ebravo-br/cicd-templates` **público**.
- **Opção B**: manter privado, mas só funciona se as duas orgs estiverem na **mesma Enterprise** com `Settings → Actions → Access` configurado.

### Limitações cross-org conhecidas

- **Não dá para linkar repos da `tramar-br` ao project "Ebravo Projetos"** via `linkProjectV2ToRepository` — o GitHub bloqueia ("Only projects owned by the same owner as the repository can be linked"). Logo, a tela "Create new issue" do project nunca lista repos da `tramar-br`. Workaround: criar a issue direto no repo (`github.com/tramar-br/<repo>/issues/new`); `issue-epic.yml` adiciona automaticamente ao project.

- **GitHub Apps não funcionam para esta automação cross-org**: tokens de App são single-installation. A chamada `addProjectV2ItemById` precisa de um único token que veja issue + project (em orgs diferentes), o que só PAT scoped no usuário consegue.
