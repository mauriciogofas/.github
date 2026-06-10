# mauriciogofas/.github — Reusable Workflows

Repositório central de workflows reutilizáveis para todos os repositórios da conta [`mauriciogofas`](https://github.com/mauriciogofas).

Dois workflows estão disponíveis: geração automática de changelog e sincronização de issues com o GitHub Projects.

---

## Workflows disponíveis

### 1. `changelog.yml` — Auto-generate changelog

Gera e commita automaticamente o arquivo `changelog.md` na branch `master` do repositório que o chama, usando a action [`charmixer/auto-changelog-action@v1.2`](https://github.com/charmixer/auto-changelog-action).

O changelog é construído a partir de releases e issues fechadas. As datas são formatadas no padrão `DD/MM/YYYY`.

**Como usar no seu repositório:**

Crie `.github/workflows/changelog.yml` com o conteúdo abaixo:

```yaml
name: Auto generate changelog

on:
  release:
    types: [published]
  issues:
    types: [closed, edited]

jobs:
  changelog:
    uses: mauriciogofas/.github/.github/workflows/changelog.yml@master
    secrets: inherit
```

**Requisitos:**

- O repositório chamador precisa ter um secret chamado `PROJECT_TOKEN` configurado, com um PAT com escopos `repo` e `project`.
- O `PROJECT_TOKEN` deve ser adicionado em **Settings > Secrets and variables > Actions** do repositório (ou herdado de organização/conta).
- A branch padrão deve ser `master`.

**O que o workflow faz:**

1. Faz checkout completo da branch `master` com histórico total (`fetch-depth: 0`).
2. Executa `charmixer/auto-changelog-action@v1.2` gerando o arquivo `changelog.md`.
3. Commita e faz push de `changelog.md` para `master` caso haja alterações.

**Observação sobre tags com `/` no nome:**

A action `charmixer/auto-changelog-action` quebra se existirem tags órfãs com `/` no nome (ex: `refs/tags/alguma/coisa`). Antes de rodar o workflow pela primeira vez, verifique e remova tags problemáticas:

```bash
# Listar tags com /
git tag | grep /

# Deletar tag local e remota
git tag -d "nome/da/tag"
git push origin --delete "nome/da/tag"
```

---

### 2. `project-sync.yml` — Sync GitHub Project

Adiciona issues ao **Project 4 "Gofas Software"** (`PVT_kwHOAHMMY84BZclp`) e atualiza o campo **Status** automaticamente conforme a ação:

| Ação no issue | Status no Project |
|---|---|
| `opened` / `reopened` (padrão) | Realizar |
| `reopened` | Em andamento |
| `closed` | Realizado |

**Como usar no seu repositório:**

Crie `.github/workflows/project-sync.yml` com o conteúdo abaixo:

```yaml
name: Sync GitHub Project

on:
  issues:
    types: [opened, closed, reopened]

jobs:
  sync:
    uses: mauriciogofas/.github/.github/workflows/project-sync.yml@master
    secrets: inherit
    with:
      issue_url: ${{ github.event.issue.html_url }}
      issue_state: ${{ github.event.issue.state }}
      issue_action: ${{ github.event.action }}
```

**Requisitos:**

- Secret `PROJECT_TOKEN` com escopos `repo` e `project` configurado no repositório chamador.
- O PAT precisa ter acesso de leitura/escrita ao Project 4.

**O que o workflow faz:**

1. Resolve o `node_id` do issue via GraphQL.
2. Adiciona o item ao Project 4 via mutation `addProjectV2ItemById`.
3. Atualiza o campo Status conforme a ação (`opened` → Realizar, `reopened` → Em andamento, `closed` → Realizado).
4. Move o item para o topo do board (`updateProjectV2ItemPosition` com `afterId: null`).

---

## Configuração do secret `PROJECT_TOKEN`

O `PROJECT_TOKEN` é um PAT (Personal Access Token) com escopos:

- `repo` — acesso aos repositórios
- `project` — acesso de leitura/escrita ao GitHub Projects

**Adicionar em um repositório individual:**

1. Acesse **Settings > Secrets and variables > Actions** no repositório.
2. Clique em **New repository secret**.
3. Nome: `PROJECT_TOKEN`, valor: o PAT gerado.

**Herdar via `secrets: inherit`:**

Se o PAT estiver configurado como secret na conta ou organização, o uso de `secrets: inherit` no workflow chamador propaga automaticamente — não é necessário configurar em cada repositório individualmente.

---

## Caller de changelog deste próprio repositório

Este repositório também roda o changelog sobre si mesmo. O workflow está em `.github/workflows/run-changelog.yml`.

---

## Estrutura do repositório

```
.github/
  workflows/
    changelog.yml       # Workflow reutilizável — gera changelog.md
    project-sync.yml    # Workflow reutilizável — sincroniza issues com Project 4
    run-changelog.yml   # Caller local — roda o changelog neste próprio repo
README.md
changelog.md            # Gerado automaticamente pelo workflow
```

---

## Referências

- [charmixer/auto-changelog-action](https://github.com/charmixer/auto-changelog-action)
- [GitHub Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [GitHub Projects GraphQL API](https://docs.github.com/en/graphql/reference/mutations#addprojectv2itembyid)
- [workflow_call — passando inputs](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onworkflow_callinputs)
