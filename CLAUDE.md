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
