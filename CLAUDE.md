# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`pipeline-as-code` — repositório `ebravo-br/cicd-templates` com workflows reutilizáveis de CI/CD para as orgs `ebravo-br` e `tramar-br`.

## Versionamento das tags — leia antes de mexer

Toda tag é fixa: aponta para um commit e não se move. Tag nova leva tudo o que está na `main` naquele momento.

| Tag | Commit | O que trouxe |
|---|---|---|
| `v8` | `b49fe6f` | `migrate_check` no `ci.yml`: migrations aplicadas num MySQL descartável e comparadas com o schema a cada push (#24) |
| `v7` | `dde60f3` | `migrate` no `cd.yml` (migration com a imagem nova antes de trocar a versão) e checagem de migration destrutiva (#23) |
| `v6` | `0e7d27c` | Sem `gke`: `deploy_platform` só com `ec2` e `ecs` (#21, #22) |
| `v5` | `63e734f` | Épico `HELPDESK`; `create-repository` no padrão atual (#18) |
| `v4` | `1a7b7cc` | `PROJECT_PAT` no `issue-epic` para projeto cross-org (#16) |
| `v3` | `7a240d9` | `context` e `dockerfile` declaráveis no `pipeline.yml` de monorepo (#14) |
| `v2` | `50fbd4d` | Deploy por ECS; OIDC quando `AWS_DEPLOY_ROLE_ARN` existe |
| `v1` | `95948a0` | Deploy por SSH no EC2; access key estática. Compartilhada pela `ebravo-br` e pela `tramar-br` |

Em 2026-11-18, o `ebcare` usa a `v8` e os outros 14 repositórios com deploy usam a `v6`. Quem usa o quê, e por quê, fica em `ebravo-br/ebravo-infra`, `docs/contexto/esteira-cicd.md`.

**Não mova a `v1`.** Ela é compartilhada pela `ebravo-br` e pela `tramar-br`, e um `git tag -f v1` altera o deploy de todas as empresas no mesmo instante, sem revisão e sem aviso. O hábito antigo era exatamente esse (`git tag -f v1 HEAD && git push --force origin v1`); ele funcionava porque só havia um caminho de deploy.

Mudança de comportamento entra em tag nova. Cada repositório migra trocando o `@vN` dos seus stubs, e voltar para a tag anterior é o caminho de volta.

Contexto e plano completos em `ebravo-br/ebravo-infra` (V8 em `docs/05`, Fase 4 em `docs/03`).

## Estrutura de workflows

| Arquivo | Descrição |
|---|---|
| `cd.yml` | Build + bump de versão + ECR + migration de banco (opcional) + deploy por SSH no EC2 ou por ECS |
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

### Migration de banco (`migrate`)

Campo opcional por componente no `.github/pipeline.yml` (no bloco do componente, em monorepo; na raiz, em repo de componente único). Sem o campo, o deploy é o de sempre.

```yaml
api:
  componente: ebcare-api
  tecnologia: nodejs
  migrate: npx prisma migrate deploy
```

O `cd.yml` roda o comando **com a imagem nova, antes de a versão nova subir**, e para o deploy se ele sair com código diferente de 0. A versão anterior continua no ar.

- **EC2:** `docker-compose run --rm --no-deps` do próprio serviço, no host do ambiente. Herda a rede e o `env_file` do compose, não publica portas e convive com o container no ar.
- **ECS:** `run-task` avulsa com a revisão nova da task definition (mesma imagem, segredos e rede do serviço), `launch-type FARGATE` mesmo onde o serviço usa Spot. Só depois de ela sair com 0 vem o `update-service`. O log fica no log group do serviço; o resumo do job mostra o stream.

Duas regras para quem escreve a migration:

1. **Compatível com o código anterior.** Durante a troca, a versão antiga roda contra o banco já migrado. Adicionar numa entrega, remover em outra.
2. **Rollback não desfaz migration.** O `rollback.yml` troca só a imagem.

O comando precisa existir **na imagem final**: CLI de migration em `dependencies`, não em `devDependencies`, quando o Dockerfile instala com `--omit=dev`.

#### Checagem das migrations no CI (`migrate_check`)

Campo opcional, ao lado do `migrate`. Com ele, o `ci.yml` ganha o job **Migrations**: sobe um MySQL 8.0 descartável no próprio runner, entrega `DATABASE_URL` e `SHADOW_DATABASE_URL` vazios e roda o comando no contexto do componente. Saiu diferente de 0, o push fica vermelho. Não toca homologação nem produção.

```yaml
api:
  migrate: npx prisma migrate deploy
  migrate_check: >-
    npx prisma migrate deploy &&
    npx prisma migrate diff --from-migrations prisma/migrations
    --to-schema-datamodel prisma/schema.prisma
    --shadow-database-url "$SHADOW_DATABASE_URL" --exit-code
```

O primeiro comando pega migration que não aplica do zero; o segundo, schema alterado sem a migration correspondente. Sem o campo, nenhum job a mais roda.

#### Checagem de migration destrutiva

Para componente com `migrate`, o primeiro passo do job de deploy lista os comandos que apagam ou alteram dado nas migrations que ainda não foram para produção. Lê os `.sql` em `.../migration(s)/...` do componente que mudaram desde a última release de produção dele (`{app}/{versao}` em monorepo).

| Pega | Não pega (o Prisma gera o tempo todo, e não apaga dado) |
|---|---|
| `DROP TABLE`/`DATABASE`/`VIEW`, `DROP [COLUMN] col`, `TRUNCATE`, `DELETE FROM`, `UPDATE ... SET`, `MODIFY`/`CHANGE` de coluna, `RENAME` | `DROP INDEX`/`KEY`/`FOREIGN KEY`/`PRIMARY KEY`/`CONSTRAINT`/`DEFAULT`, `ON DELETE`/`ON UPDATE`, comentários |

Com achado, o deploy **para**, antes do bump, do build e da migration, até ser rodado de novo com o input `confirmar_migration_destrutiva` marcado. Os achados aparecem como anotação no job e numa tabela no resumo, e a confirmação registra quem confirmou.

- **Produção:** sempre ligada.
- **Homologação:** desligada. Só lista, a não ser que o repositório tenha a variável `MIGRATION_GATE_HOMOL=true` (para ensaio).

O stub precisa declarar e repassar o input para poder confirmar. Sem ele, um deploy de produção com migration destrutiva não passa, e o erro explica:

```yaml
# deploy.yml (stub), em workflow_dispatch.inputs
      confirmar_migration_destrutiva:
        description: 'Só produção: confirmo que revisei as migrations destrutivas listadas'
        type: boolean
        default: false
# e em jobs.cd.with
      confirmar_migration_destrutiva: ${{ inputs.confirmar_migration_destrutiva }}
```

O porquê: o Prisma não tem migration de volta, o MySQL confirma DDL comando a comando, e o Aurora é compartilhado por todas as aplicações. Um `DROP` aplicado só volta restaurando backup num cluster à parte e reimportando o dado.

No ECS, a role de deploy precisa de `ecs:RunTask` (ver `ebravo-infra`, módulo `cicd-oidc`).

Para consolidar um par `*-api` + `*-web` num monorepo (histórico preservado, esteira monorepo, herança de permissões, arquivamento das origens), use a skill **`migrate-to-monorepo`** (`.claude/skills/migrate-to-monorepo/`).

### Monorepos migrados

| Monorepo | Origens (arquivadas) | Stack |
|---|---|---|
| `tramar-br/tramar-tps` | `tramar-tps-api` + `tramar-tps-web` | Spring Boot + Angular |
| `tramar-br/tramar-pedidos` | `tramar-pedidos-api` + `tramar-pedidos-web` | Spring Boot + Angular |
| `ebravo-br/ers` | `ers-api` + `ers-web` | Spring Boot + Angular |
| `ebravo-br/cemiterio-campo-limpo` | `cemiterio-campo-limpo-api` + `cemiterio-campo-limpo-web` | Spring Boot + Angular |
| `ebravo-br/ebravo-atos` | `ebravo-atos-api` + `ebravo-atos-web` | Spring Boot + Angular |
| `ebravo-br/ebdocs` | `ebdocs-api` + `ebdocs-web` | Spring Boot + Angular |
| `ebravo-br/datapat` | `datapat-api` + `datapat-web` | Spring Boot + Angular |
| `ebravo-br/ebcare` | (nasceu monorepo) | NestJS + Next.js |

Os repos de origem `tramar-*-api`/`tramar-*-web` foram arquivados após a migração (histórico preservado dentro do monorepo em `apps/api` e `apps/web`).

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

## Skills como plugin do Claude Code

As skills de `.claude/skills/` são a **fonte única** das skills de organização. Este repositório também é um marketplace de plugin (`.claude-plugin/marketplace.json`), cujo plugin `ebravo` tem a própria pasta `.claude/` como raiz. Outros repositórios, como o `ebravo-infra`, declaram o plugin no `.claude/settings.json` e recebem as skills como `/ebravo:create-repository`, `/ebravo:add-techlead` e `/ebravo:migrate-to-monorepo`, sem cópia.

Mudou uma skill? Basta o merge aqui. Quem usa o plugin recebe a versão nova no `claude plugin update ebravo@ebravo-br` (ou na atualização automática do marketplace). Não precisa de tag da esteira.

