---
name: add-techlead
description: Concede acesso completo de tech lead a um usuário nas orgs ebravo-br e tramar-br — garante membership da org e o adiciona ao team `techleads` como maintainer (que herda a organization role `all_repo_admin`, admin em todos os repos sem poderes de Owner). Use quando pedirem para "adicionar um techlead", "dar acesso de admin nos repos a fulano", "tornar fulano tech lead".
---

# add-techlead

Concede a um usuário o acesso padrão de tech lead nas orgs `ebravo-br` e `tramar-br`.

## O que esse acesso significa

- **Membro** da org (não Owner — sem billing/membros/settings da org).
- **Maintainer** do team `techleads` em cada org.
- Como o team `techleads` tem a organization role `all_repo_admin` (id `8136`, `base_role: admin`), o usuário **herda admin em todos os repositórios** automaticamente.
- Ser **maintainer** (e não só `member`) é o que faz aparecer, na aba *Repositories* do team, o dropdown de **Role** e o botão **Add repository**. Membro comum vê a tela read-only.

## Pré-requisitos

- `gh` autenticado com um usuário Owner das duas orgs (ou com o `PROJECT_PAT`, que tem `read:org` + admin suficiente).
- O usuário-alvo precisa ter uma conta no GitHub. Se ainda não for membro da org, o `PUT /memberships` envia um **convite** que ele precisa aceitar antes de herdar os acessos.

## Passos

Receba o `login` do GitHub do usuário-alvo. Para **cada** org em `ebravo-br tramar-br`:

```bash
USER="<login-do-usuario>"
for org in ebravo-br tramar-br; do
  echo "=== $org: garantindo membership de $USER ==="
  gh api -X PUT /orgs/$org/memberships/$USER -f role=member --jq '{state, role}'

  echo "=== $org: adicionando $USER ao techleads como maintainer ==="
  gh api -X PUT /orgs/$org/teams/techleads/memberships/$USER -f role=maintainer --jq '{role, state}'
done
```

> A organization role `all_repo_admin` **não precisa ser atribuída ao usuário** — ela já está no team `techleads`. Basta entrar no team.

## Verificação

```bash
USER="<login-do-usuario>"
for org in ebravo-br tramar-br; do
  echo "=== $org ==="
  echo -n "  membership org: "; gh api /orgs/$org/memberships/$USER --jq '.state + " / " + .role'
  echo -n "  role no team:   "; gh api /orgs/$org/teams/techleads/memberships/$USER --jq '.role + " (" + .state + ")"'
  echo -n "  tem all_repo_admin: "; gh api /orgs/$org/organization-roles/8136/users --jq "[.[].login] | index(\"$USER\") != null"
done
```

Esperado em cada org: `active / member`, `maintainer (active)`, `true`.

## Notas

- Se `state` aparecer como `pending`, o usuário ainda não aceitou o convite da org — os acessos só valem depois que ele aceitar.
- Maintainer do team pode gerenciar **membros do team** `techleads` (inclusive conceder esse mesmo acesso a outros). Não é poder de Owner da org.
- Para **remover** um tech lead: `gh api -X DELETE /orgs/$org/teams/techleads/memberships/$USER` (remove do team e, com isso, o `all_repo_admin`). Remover da org é `DELETE /orgs/$org/memberships/$USER`.
