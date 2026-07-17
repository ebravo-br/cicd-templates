# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`pipeline-as-code` — repositório `ebravo-br/cicd-templates` com workflows reutilizáveis de CI/CD para as orgs `ebravo-br` e `tramar-br`. A tag `v1` é apontada manualmente após cada push (`git tag -f v1 HEAD && git push --force origin v1`).

## Estrutura de workflows

| Arquivo | Descrição |
|---|---|
| `cd.yml` | Build + bump de versão + ECR + deploy SSH |
| `ci.yml` | Build only (sem Docker/ECR) |
| `rollback.yml` | Rollback para tag anterior no ECR |
| `issue-epic.yml` | Adiciona issue ao projeto e define Épico/Status |
| `pr-status.yml` | Muda Status da issue para "Code Review" ao abrir PR |
| `project-automation.yml` | Fecha branch e move issue ao dar merge no PR |
| `branch-naming.yml` | Renomeia branch `{N}-{slug}` para `{tipo}/{N}` após criação |

## Gitflow — convenção de branches

| Tipo | Prefixo |
|---|---|
| Feature | `feature/{issue}` |
| Bug | `bug/{issue}` |
| Task | `task/{issue}` |

Branches criadas pelo GitHub via "Create a branch for this issue" (`{N}-{slug}`) são renomeadas automaticamente pelo workflow `branch-naming.yml`.

## Formato de versão (cd.yml)

### Homologação

```
{versao_atual}-B{run_number}-{tipo_letra}{numero_issue}
```

| Campo | Significado |
|---|---|
| `{versao_atual}` | Versão base extraída do arquivo de versão (ex: `1.52.0`) |
| `B{run_number}` | `B` + número sequencial do workflow run (`github.run_number`) |
| `F{N}` | Feature — branch `feature/{N}` |
| `B{N}` | Bug — branch `bug/{N}` |
| `T{N}` | Task — branch `task/{N}` |

Exemplos:
- `1.52.0-B1-F1`   → Build 1, Feature issue 1
- `1.52.0-B2-B55`  → Build 2, Bug issue 55
- `1.52.0-B34-T1`  → Build 34, Task issue 1
- `1.52.0-B199-T88`→ Build 199, Task issue 88

Fallback (branch sem padrão): `{versao}-B{run_number}`

### Produção

Bump do minor: `1.52.0` → `1.53.0`

### Tag/Release de produção

| Tipo | Formato da tag | Exemplo |
|---|---|---|
| Single-repo | `{versao}` | `1.53.0` |
| Monorepo | `{app}/{versao}` | `api/1.97.0`, `web/1.92.0` |

O prefixo (`api`/`web`) sai de `matrix.app`. A imagem no ECR usa sempre `{componente}:{versao}` (sem prefixo). Homologação não cria tag/release.

## Monorepo

Os reusable `ci.yml` / `cd.yml` / `rollback.yml` são **monorepo-aware**. Com `monorepo: true` no `.github/pipeline.yml`, o job `setup` itera as chaves `api`/`web` e resolve por app: `context: apps/<app>`, `version_file: apps/<app>/{pom.xml|package.json|pyproject.toml}` e `dockerfile: infra/docker/<app>/Dockerfile`. `cd.yml`/`rollback.yml` aceitam `deploy_apps: ambos | api | web`. Em produção, a tag/release de cada app é prefixada: `{app}/{versao}` (ex.: `api/1.97.0`, `web/1.92.0`).

Layout esperado (modelo `ebravo-br/ebcare`):

```
apps/api/            # código do backend (versão no pom.xml/package.json)
apps/web/            # código do frontend
infra/docker/api/Dockerfile
infra/docker/web/Dockerfile
.github/pipeline.yml # monorepo: true + blocos api/web (componente = nome da imagem ECR)
```

`.github/pipeline.yml` de monorepo:

```yaml
monorepo: true
api:
  componente: <repo>-api
  tecnologia: spring-boot-java
web:
  componente: <repo>-web
  tecnologia: angular
  node: '20'
```

Para consolidar um par `*-api` + `*-web` num monorepo (histórico preservado, esteira monorepo, herança de permissões, arquivamento das origens), use a skill **`migrate-to-monorepo`** (`.claude/skills/migrate-to-monorepo/`). Exemplo aplicado: `tramar-br/tramar-tps` (consolidou `tramar-tps-api` Spring Boot + `tramar-tps-web` Angular).

## Auth / secrets

- `PROJECT_PAT` — PAT clássico do usuário `juniorebravo` (scopo: `project`, `repo`, `read:org`). Necessário para automações cross-org (ebravo-br ↔ tramar-br). Configurado como org secret em ambas as orgs.
- `GH_TOKEN_BUMP` — mesmo PAT, usado pelo deploy para commitar o bump de versão diretamente na branch protegida (tem `bypass_pull_request_allowances` em todos os repos).
- `enforce_admins: true` está ativo em todos os repos. Para push direto na main do cicd-templates, desabilitar temporariamente via `gh api -X DELETE /repos/ebravo-br/cicd-templates/branches/main/protection/enforce_admins`.

## Organization roles

Para conceder **admin de todos os repositórios sem poderes de Owner** (billing, membros, settings da org), usa-se a organization role predefinida `all_repo_admin` (id `8136` em ambas as orgs), atribuída ao team `techleads`. O usuário continua `Member` da org — não promover a Owner.

- Atribuir: `gh api -X PUT /orgs/{org}/organization-roles/teams/techleads/8136` (resposta `204`).
- Verificar: `gh api /orgs/{org}/organization-roles/8136/teams --jq '.[].slug'` (deve listar `techleads`).
- Remover: `gh api -X DELETE /orgs/{org}/organization-roles/teams/techleads/8136`.

Aplicado em `ebravo-br` e `tramar-br`. Como a role está no team, qualquer membro adicionado ao `techleads` herda o acesso automaticamente.

**Member vs maintainer do team:** o `all_repo_admin` dá admin nos repositórios individualmente, mas para gerenciar a aba *Repositories* do próprio team (dropdown de Role + botão "Add repository") o usuário precisa ser **maintainer** do `techleads`, não só `member`. Adicionar/promover: `gh api -X PUT /orgs/{org}/teams/techleads/memberships/{user} -f role=maintainer`.

Para conceder o pacote completo de tech lead a um novo usuário (membership + maintainer no techleads nas duas orgs), use a skill **`add-techlead`** (`.claude/skills/add-techlead/`).
