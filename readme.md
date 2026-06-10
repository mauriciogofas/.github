# Reusable Workflows — Auto Changelog & GitHub Projects Sync

Dois workflows reutilizáveis para qualquer conta ou organização GitHub:

1. **`changelog.yml`** — gera e commita `changelog.md` automaticamente a cada release ou issue fechada
2. **`project-sync.yml`** — adiciona issues a um GitHub Project V2 e move entre colunas conforme o status

---

## Como funciona

### Changelog

A cada release publicada ou issue fechada/editada, o workflow chama [`charmixer/auto-changelog-action@v1.2`](https://github.com/charmixer/auto-changelog-action), gera o `changelog.md` a partir do histórico de releases e issues do repositório e commita diretamente na branch `master`.

### Project Sync

A cada issue aberta, reaberta ou fechada, o workflow:

1. Resolve o `node_id` da issue via GraphQL
2. Adiciona a issue ao projeto via `addProjectV2ItemById`
3. Atualiza o campo **Status** conforme a ação:
   - `opened` → **Realizar**
   - `reopened` → **Em andamento**
   - `closed` → **Realizado**
4. Move o item para o topo do board via `updateProjectV2ItemPosition(afterId: null)`

---

## Pré-requisitos

### 1. Criar o projeto com as três colunas

No GitHub, acesse **Projects > New project > Board**.

Adicione três colunas (com estes nomes, ou os nomes que preferir — você vai mapear os IDs logo abaixo):

- **Realizar** — issues a fazer
- **Em andamento** — issues em progresso
- **Realizado** — issues concluídas

As issues se movem automaticamente entre essas colunas via workflow conforme são abertas, reabertas ou fechadas. Nenhuma ação manual é necessária após a configuração.

### 2. Obter o ID do projeto e os IDs das colunas

Execute as queries abaixo com o GitHub CLI autenticado com um PAT com escopos `repo` + `project`:

```bash
# Listar seus projetos e obter o ID (formato PVT_...)
gh api graphql -f query='
query {
  user(login: "SEU_USUARIO") {
    projectsV2(first: 10) {
      nodes { number title id }
    }
  }
}'
```

Com o ID do projeto em mãos, obtenha os campos e opções:

```bash
gh api graphql -f query='
query {
  node(id: "PVT_SEU_PROJETO_ID") {
    ... on ProjectV2 {
      fields(first: 20) {
        nodes {
          ... on ProjectV2SingleSelectField {
            id
            name
            options { id name }
          }
        }
      }
    }
  }
}'
```

A resposta terá um campo chamado **Status** (ou o nome que você escolheu). Anote:
- O `id` do campo (formato `PVTSSF_...`) — este é o `STATUS_FIELD`
- O `id` de cada opção (formato hexadecimal curto, ex: `f75ad846`) — estes são os `STATUS_REALIZAR`, `STATUS_EM_ANDAMENTO`, `STATUS_REALIZADO`

### 3. Criar o PAT `PROJECT_TOKEN`

Crie um Personal Access Token com escopos:
- `repo` — acesso aos repositórios
- `project` — leitura e escrita no GitHub Projects V2

Adicione como secret em cada repositório que usar os workflows:
**Settings > Secrets and variables > Actions > New repository secret**
Nome: `PROJECT_TOKEN`

Se preferir não configurar em cada repo individualmente, adicione o secret na conta ou organização — o `secrets: inherit` nos callers vai propagá-lo automaticamente.

---

## Configurar o workflow reutilizável (este repositório)

No arquivo `.github/workflows/project-sync.yml` deste repositório, substitua os valores:

```yaml
PROJECT_ID="PVT_SEU_PROJETO_ID"
STATUS_FIELD="PVTSSF_SEU_CAMPO_ID"
STATUS_REALIZAR="ID_DA_OPCAO_REALIZAR"
STATUS_EM_ANDAMENTO="ID_DA_OPCAO_EM_ANDAMENTO"
STATUS_REALIZADO="ID_DA_OPCAO_REALIZADO"
```

---

## Adicionar aos seus repositórios

Crie dois arquivos em cada repositório que quiser integrar:

**`.github/workflows/changelog-caller.yml`**
```yaml
name: Auto generate changelog

on:
  release:
    types: [published]
  issues:
    types: [closed, edited]

jobs:
  changelog:
    uses: SEU_USUARIO/.github/.github/workflows/changelog.yml@master
    secrets: inherit
```

**`.github/workflows/project-sync-caller.yml`**
```yaml
name: Sync GitHub Project

on:
  issues:
    types: [opened, closed, reopened]

jobs:
  sync:
    uses: SEU_USUARIO/.github/.github/workflows/project-sync.yml@master
    secrets: inherit
    with:
      issue_url: ${{ github.event.issue.html_url }}
      issue_state: ${{ github.event.issue.state }}
      issue_action: ${{ github.event.action }}
```

Substitua `SEU_USUARIO` pelo seu usuário ou organização GitHub.

---

## Observações importantes

**`github.event` não propaga em `workflow_call`**

Ao usar `workflow_call`, o contexto `github.event` não é passado automaticamente para o workflow chamado. Por isso o caller passa `issue_url`, `issue_state` e `issue_action` explicitamente via `with:`. O workflow reutilizável os recebe como `inputs`. Nunca tente acessar `github.event.issue.*` dentro do workflow reutilizável — estará vazio.

**`PROJECT_TOKEN` deve existir no repo caller**

Secrets do repositório hospedeiro (`.github`) não propagam para repositórios externos. O `PROJECT_TOKEN` precisa estar configurado no repositório que invoca o workflow (ou herdado de conta/organização via `secrets: inherit`).

**Tags com `/` no nome quebram o auto-changelog**

A action `charmixer/auto-changelog-action` falha se existirem tags com `/` no nome. Antes de usar o workflow pela primeira vez, verifique:

```bash
git tag | grep /
```

Para remover:
```bash
git tag -d "nome/da/tag"
git push origin --delete "nome/da/tag"
```

**`changelog.md` é gerado em letras minúsculas**

O nome do arquivo de saída gerado pela action é sempre `changelog.md` (minúsculo). Configure `output: changelog.md` no workflow.

**Ordem correta ao publicar uma release**

Feche todos os issues relevantes primeiro, aguarde o workflow de changelog concluir e só então crie a release. Se criar a release antes de fechar os issues, eles aparecerão em uma seção "Unreleased" no changelog. Para corrigir: reabra e feche novamente o issue para retriggar o workflow.

---

## Estrutura deste repositório

```
.github/
  workflows/
    changelog.yml          # Workflow reutilizável — gera changelog.md
    project-sync.yml       # Workflow reutilizável — sincroniza issues com projeto
    run-changelog.yml      # Caller local — roda o changelog neste próprio repo
README.md
changelog.md               # Gerado automaticamente
```
