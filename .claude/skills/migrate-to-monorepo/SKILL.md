---
name: migrate-to-monorepo
description: Migra um par de repositórios *-api / *-web para um único monorepo (padrão ebravo-br/ebcare), preservando o histórico de commits de ambos, com a esteira CI/CD monorepo-aware de ebravo-br/cicd-templates, herdando as permissões/branch-protection dos repos de origem e arquivando-os ao final. Use quando pedirem para "migrar multirepo para monorepo", "juntar api e web num repo só", "consolidar tramar-*-api e tramar-*-web", ou via /migrate-to-monorepo <monorepo> <org> <api-repo> <web-repo>.
---

# Migrate to Monorepo

Consolida um par `*-api` (backend) + `*-web` (frontend) num único monorepo seguindo o
padrão do `ebravo-br/ebcare`: código em `apps/api` e `apps/web`, Dockerfiles em
`infra/docker/{api,web}`, `.github/pipeline.yml` com `monorepo: true`, e wrappers finos
que chamam os reusable workflows de `ebravo-br/cicd-templates@v1`.

**Achado-chave:** os reusable `ci.yml` / `cd.yml` / `rollback.yml` já são monorepo-aware.
Quando `monorepo: true`, o job `setup` itera as chaves `api`/`web` do `pipeline.yml` e resolve
por app: `context: apps/<app>`, `version_file: apps/<app>/{pom.xml|package.json|pyproject.toml}`
e `dockerfile: infra/docker/<app>/Dockerfile`. O `cd.yml`/`rollback.yml` aceitam
`deploy_apps: ambos | api | web`. **A esteira não muda — só o layout do repo destino.**

## Input: $ARGUMENTS

Formato: `<monorepo> <org> <api-repo> <web-repo>`

Exemplos:
- `/migrate-to-monorepo tramar-tps tramar-br tramar-tps-api tramar-tps-web`
- `/migrate-to-monorepo tramar-pedidos tramar-br tramar-pedidos-api tramar-pedidos-web`

Parâmetros:
- **monorepo**: nome do repo destino (sem org prefix). Criado em `<org>/<monorepo>`.
- **org**: org GitHub (`ebravo-br` ou `tramar-br`).
- **api-repo / web-repo**: nomes dos repos de origem (na mesma `<org>`).

## Pré-requisitos

- `git-filter-repo` instalado (`brew install git-filter-repo`).
- `gh` autenticado como owner/admin da org (para criar repo, teams, branch protection, arquivar).
- **Cross-org (org = tramar-br):** a secret `PROJECT_PAT` precisa existir em `tramar-br` (mesmo
  value de `ebravo-br`) — automação de projeto cross-org exige PAT explícito, não `secrets: inherit`.

## Decisões a confirmar com o usuário (antes de executar)

1. **Histórico:** preservar (via `git filter-repo`, recomendado) ou snapshot limpo.
2. **Repos de origem:** arquivar (recomendado), manter ou deletar.
3. **Legado/sensíveis:** confirmar remoção de lixo e de segredos versionados (wallets, credenciais).
4. **Permissões:** o monorepo herda teams/colaboradores/branch-protection dos originais — como
   conceder acesso é sensível, **confirmar a lista exata** (o classificador de segurança pode
   bloquear grants "derivados de output"; apresentar os grants nomeados e pedir confirmação).

---

## Instruções

Trabalhar num diretório temporário (scratchpad). Todos os comandos abaixo usam placeholders
`<monorepo>`, `<org>`, `<api-repo>`, `<web-repo>`.

### Passo 0 — Pré-checagem

```bash
gh repo view <org>/<monorepo> 2>/dev/null && echo "ABORTAR: destino já existe" && exit 1
gh repo view <org>/<api-repo> --json name >/dev/null   # deve existir
gh repo view <org>/<web-repo> --json name >/dev/null   # deve existir
git filter-repo --version >/dev/null || echo "instalar git-filter-repo"
```

Levantar dos repos de origem (para replicar depois): tecnologia e arquivo de versão, teams e
permissões, colaboradores diretos, e a branch protection da `main`:

```bash
gh api repos/<org>/<api-repo>/teams --jq '.[] | {slug, permission}'
gh api "repos/<org>/<api-repo>/collaborators?affiliation=direct" --jq '.[] | {login, role_name}'
gh api repos/<org>/<api-repo>/branches/main/protection    # capturar enforce_admins, reviews, restrictions, bypass
```

### Passo 1 — Consolidar com histórico preservado

```bash
git clone https://github.com/<org>/<api-repo>.git
git clone https://github.com/<org>/<web-repo>.git
( cd <api-repo> && git filter-repo --force --to-subdirectory-filter apps/api )
( cd <web-repo> && git filter-repo --force --to-subdirectory-filter apps/web )

git init -b main <monorepo> && cd <monorepo>
git commit --allow-empty -m "chore: inicializa monorepo <monorepo>"
git remote add api ../<api-repo> && git remote add web ../<web-repo>
git fetch api main && git fetch web main
git merge --no-edit --allow-unrelated-histories api/main -m "chore: importa <api-repo> em apps/api (histórico preservado)"
git merge --no-edit --allow-unrelated-histories web/main -m "chore: importa <web-repo> em apps/web (histórico preservado)"
git remote remove api && git remote remove web
```

### Passo 2 — Limpar legado e sensíveis (commits dedicados)

Remover `.github/` de cada app (o monorepo usa um `.github/` na raiz):
```bash
git rm -r apps/api/.github apps/web/.github
```

Padrões de lixo a remover (ajustar ao que existir):
- **Java/Maven (api):** `pom.xml.versionsBackup`, `*_bkp`, `jenkins.properties`, `package-lock.json` (resíduo).
- **Node/Angular (web):** `.travis.yml`, `bitbucket-pipelines.yml`, `jenkins.properties`, `*.bak`,
  Dockerfiles alternativos na raiz (`Dockerfile.homol`, `DockerfileLocal`, …), `nbproject/`, `.vscode/` (opcional).
- **Sensíveis:** wallets versionadas (`*/oracle_cloud_wallet/`), credenciais em texto plano em
  `docker-compose.yml` (parametrizar com `${VAR}`).

**Binários grandes no histórico** (jars, zips) incham o `.git`. Detectar e purgar por caminho
exato (só os que NÃO estão no HEAD, para não apagar arquivos vivos):
```bash
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob"{print $3,$4}' | sort -rn | head -15   # inspecionar maiores blobs
git filter-repo --force --invert-paths --path <caminho/exato/arquivo.jar> --path <outro.zip>
```
> Use `--path <arquivo-exato>` para binários dentro de pastas que você quer preservar (ex.:
> `apps/api/deploy/x.jar` sem apagar os `.sql` de `apps/api/deploy/`). Só use `--path <dir>` para
> pastas history-only inteiras. Confirme antes que o caminho não está no HEAD.

### Passo 3 — Reestruturar infra para o layout monorepo

```bash
mkdir -p infra/docker/api infra/docker/web infra/compose
git mv apps/api/infra/docker/Dockerfile infra/docker/api/Dockerfile
git mv apps/web/infra/docker/Dockerfile infra/docker/web/Dockerfile
git mv apps/api/infra/compose/docker-compose.yml infra/compose/docker-compose-api.yml
git mv apps/web/infra/compose/docker-compose.yml infra/compose/docker-compose-web.yml
git rm -r apps/api/infra apps/web/infra 2>/dev/null || true
```
Os Dockerfiles usam caminhos relativos ao build context (`./target/...`, `./config/nginx.conf`,
`/dist/<app>`). No monorepo o `cd.yml` builda com `context=apps/<app>` e `-f infra/docker/<app>/Dockerfile`,
então esses caminhos continuam válidos **sem edição**. Manter `apps/web/config/nginx.conf` no lugar.
(Reaproveita a lógica de mover Dockerfile/compose de `.claude/commands/setup-pipeline-tramar.md`, Passo 5.)

### Passo 4 — `.github/pipeline.yml` monorepo

Detectar tecnologia por app (`pom.xml` → `spring-boot-java`; `angular.json` → `angular`;
`package.json` sem angular → `nodejs`/`react`; `pyproject.toml` → `python`) e reaproveitar
`componente`/`node`/`jdk` dos `pipeline.yml` de origem (**manter os `componente` idênticos** para
preservar os nomes das imagens no ECR):

```yaml
monorepo: true

api:
  componente: <api-repo>
  tecnologia: spring-boot-java

web:
  componente: <web-repo>
  tecnologia: angular
  node: '<versao-do-origem>'
```

### Passo 5 — Wrappers `.github/workflows/` (6 arquivos)

Todos apontam para `ebravo-br/cicd-templates@v1`. `deploy.yml` e `rollback.yml` **precisam** do
input `deploy_apps: [ambos, api, web]` (monorepo).

**Regra cross-org de secrets:**
- **org = ebravo-br** → `secrets: inherit` (mesma org do cicd-templates).
- **org = tramar-br** → passar secrets **explicitamente** (`inherit` NÃO propaga cross-org):
  - `deploy.yml`/`rollback.yml`: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `SSH_KEY_HOMOL`, `SSH_KEY_PROD`, `GH_TOKEN_BUMP`.
  - `branch-naming.yml`/`issue-epic.yml`/`project-automation.yml`: `PROJECT_PAT: ${{ secrets.PROJECT_PAT }}`.

Arquivos: `ci.yml` (push em `main`/`feature|task|bug/**`), `deploy.yml` (`workflow_dispatch`,
inputs `target_env`/`deploy_platform`/`deploy_apps`), `rollback.yml` (idem + `image_tag`),
`branch-naming.yml` (`on: create`), `issue-epic.yml` (`on: issues opened`),
`project-automation.yml` (`on: pull_request opened/closed`). Cabeçalho `# >>> NAO EDITE ESTE ARQUIVO <<<`.
Templates completos: ver `.claude/commands/setup-pipeline-tramar.md` (Passos 2–3d) — apenas
adicionar o input/passagem de `deploy_apps` em deploy/rollback.

Adicionar um `README.md` na raiz descrevendo a estrutura. Commit final.

### Passo 6 — Criar o repo e publicar

```bash
gh repo create <org>/<monorepo> --private --description "<desc>"
gh api -X PATCH repos/<org>/<monorepo> \
  -F allow_merge_commit=true -F allow_squash_merge=true -F allow_rebase_merge=true \
  -F delete_branch_on_merge=false   # ou herdar exatamente dos originais
git remote add origin https://github.com/<org>/<monorepo>.git
git push origin main            # ANTES da branch protection
# se purgou histórico depois do 1º push: git push --force origin main
```

### Passo 7 — Herdar permissões e branch protection (dos originais)

> **Confirmar a lista com o usuário antes** (grants de acesso; o classificador pode bloquear).
> Ler a config real dos repos de origem (Passo 0) e reaplicar — não usar defaults da create-repository.

```bash
# teams (ex.: devs=push, techleads=admin — o que os originais tiverem)
gh api -X PUT orgs/<org>/teams/<slug>/repos/<org>/<monorepo> -f permission=<push|admin>
# colaboradores diretos
gh api -X PUT repos/<org>/<monorepo>/collaborators/<login> -f permission=<pull|push|admin>
# branch protection (copiar enforce_admins, reviews, dismiss, restrictions.users/teams/apps, bypass)
gh api -X PUT repos/<org>/<monorepo>/branches/main/protection --input - <<'EOF'
{ ...cópia exata da proteção dos repos de origem... }
EOF
```

Secrets: nada a fazer se são org-level. Épico: automático via `issue-epic.yml`
(`owner=tramar-br → TRAMAR`; senão pelo prefixo do nome).

### Passo 8 — Validar

```bash
gh api "repos/<org>/<monorepo>/git/trees/main?recursive=1" --jq '.tree[].path' | grep -E 'apps/(api|web)$|infra/docker/(api|web)/Dockerfile|pipeline.yml'
gh run list -R <org>/<monorepo> --limit 3       # push dispara Build
gh run view <id> -R <org>/<monorepo> --json jobs --jq '.jobs[] | "\(.conclusion) \(.name)"'
```
Esperado: matrix com um job de Build por app (api + web), cada um em `apps/<app>`.
**Não disparar `Deploy` como validação** — `cd.yml` não tem dry-run (faz bump + ECR push + SSH real).

### Passo 9 — Arquivar origens (após validar)

```bash
gh repo archive <org>/<api-repo> --yes
gh repo archive <org>/<web-repo> --yes
```

---

## Limitações / caveats

- **Deploy on-premises vs EC2:** o `cd.yml` faz SSH para hosts EC2 hardcoded. Se o par migrado
  fazia deploy on-prem manual, o primeiro deploy do monorepo pode não alcançar os hosts corretos —
  sinalizar; ajuste de host é fora do escopo da migração.
- **Cross-org project link:** repos de `tramar-br` **não** podem ser linkados ao Project "Ebravo
  Projetos" (`linkProjectV2ToRepository` é same-owner). A automação via `issue-epic.yml` adiciona a
  issue ao project por PAT mesmo assim.
- **Rewrite de histórico:** purgar binários com `filter-repo` reescreve SHAs — só fazer no monorepo
  recém-criado (antes de terceiros clonarem) e usar `git push --force`.
