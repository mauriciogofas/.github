# Reusable Workflows — Auto Changelog & GitHub Projects Sync

Dois workflows reutilizáveis para qualquer conta ou organização GitHub:

1. **`changelog.yml`** — gera e commita `changelog.md` automaticamente a cada release ou issue fechada
2. **`project-sync.yml`** — adiciona issues a um GitHub Project V2 e move entre colunas conforme o status

Todos os dados específicos do usuário (credenciais, IDs de projeto, usuário Git) ficam nos arquivos **caller** dentro do seu próprio repositório. Os workflows deste repo não armazenam, registram nem recebem nenhum dado seu além do que o próprio GitHub expõe ao runner durante a execução — o mesmo acesso que qualquer GitHub Action pública já tem.

> **Nota técnica sobre privacidade:** por comportamento padrão do GitHub Actions, o `github context` dentro de um workflow reutilizável é sempre associado ao workflow caller. Isso significa que o runner que executa o `project-sync.yml` e o `changelog.yml` deste repo tem acesso ao contexto do seu repositório (como `github.repository`) durante a execução. Esse é o comportamento padrão documentado do GitHub para todos os workflows reutilizáveis — não é exclusivo deste projeto. Nenhum dado é coletado, armazenado ou transmitido além da execução normal do runner.

---

## Como funciona

### Changelog

A cada release publicada ou issue fechada/editada, o workflow chama [`charmixer/auto-changelog-action@v1.2`](https://github.com/charmixer/auto-changelog-action), gera `changelog.md` a partir do histórico de releases e issues do repositório e commita diretamente na branch `master`.

### Project Sync

A cada issue aberta, reaberta ou fechada, o workflow:

1. Resolve o `node_id` da issue via GraphQL
2. Adiciona a issue ao seu projeto via `addProjectV2ItemById`
3. Atualiza o campo **Status** conforme a ação:
   - `opened` → coluna **Realizar**
   - `reopened` → coluna **Em andamento**
   - `closed` → coluna **Realizado**
4. Move o item para o topo do board via `updateProjectV2ItemPosition`

---

## Passo a passo para configurar

### Passo 1 — Criar o projeto no GitHub

Acesse **github.com > Projects > New project > Board**.

Crie três colunas com os nomes que preferir. A correspondência entre nome e coluna é configurada por você no caller — não há nomes obrigatórios. Sugestão:

- **Realizar** — issues a fazer
- **Em andamento** — issues em progresso
- **Realizado** — issues concluídas

As issues se moverão entre essas colunas automaticamente conforme forem abertas, reabertas ou fechadas.

---

### Passo 2 — Obter os IDs do projeto e das colunas

Você vai precisar de três valores: o ID do projeto, o ID do campo Status e os IDs de cada opção (coluna). Tudo obtido via GraphQL com o GitHub CLI.

**Primeiro, liste seus projetos para obter o ID do projeto (`PVT_...`):**

```bash
gh api graphql -f query='
query {
  user(login: "SEU_USUARIO") {
    projectsV2(first: 10) {
      nodes { number title id }
    }
  }
}'
```

A resposta terá um campo `id` no formato `PVT_kwXXXXXXXXXXXX`. Anote o ID do projeto que deseja usar.

**Depois, liste os campos do projeto para obter o ID do campo Status e das colunas (`PVTSSF_...` e os IDs hexadecimais):**

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

A resposta terá algo como:

```json
{
  "id": "PVTSSF_lAXXXXXXXXXXXXXX",
  "name": "Status",
  "options": [
    { "id": "aaaaaaaa", "name": "Realizar" },
    { "id": "bbbbbbbb", "name": "Em andamento" },
    { "id": "cccccccc", "name": "Realizado" }
  ]
}
```

Anote:
- `id` do campo Status → `PVTSSF_...` — vai para `status_field` no caller
- `id` de cada opção → vai para `status_realizar`, `status_em_andamento` e `status_realizado`

---

### Passo 3 — Criar o PAT `PROJECT_TOKEN`

Crie um Personal Access Token em **github.com > Settings > Developer settings > Personal access tokens** com os escopos:

- `repo` — acesso aos repositórios
- `project` — leitura e escrita no GitHub Projects V2

Adicione como secret no repositório onde quer usar os workflows:
**Settings > Secrets and variables > Actions > New repository secret**
Nome: `PROJECT_TOKEN`

Se preferir configurar uma única vez para todos os repos, adicione na sua conta em **Settings > Secrets and variables > Actions** e use `secrets: inherit` no caller — o secret será propagado automaticamente.

---

### Passo 4 — Criar os callers no seu repositório

Copie os arquivos `.example` deste repo como ponto de partida. Eles ficam em `.github/workflows/` e não são executados pelo GitHub Actions por terem extensão `.example`.

Crie dois arquivos no seu repositório em `.github/workflows/`:

---

#### `changelog-caller.yml`

Baseado em [`changelog-caller.yml.example`](.github/workflows/changelog-caller.yml.example):

```yaml
name: Changelog
on:
  release:
    types: [published]
  issues:
    types: [closed, edited]
jobs:
  changelog:
    permissions:
      contents: write
    uses: SEU_USUARIO/.github/.github/workflows/changelog.yml@master
    with:
      git_user: "seu-usuario-github"
      git_name: "Seu Nome"
      git_email: "seu@email.com"
    secrets: inherit
```

**Linhas a editar:**

| Linha | Campo | O que colocar |
|---|---|---|
| 11 | `uses:` | Substituir `SEU_USUARIO` pelo seu usuário GitHub |
| 13 | `git_user:` | Seu usuário GitHub (usado na URL do push) |
| 14 | `git_name:` | Seu nome para o commit |
| 15 | `git_email:` | Seu email para o commit |

---

#### `project-sync-caller.yml`

Baseado em [`project-sync-caller.yml.example`](.github/workflows/project-sync-caller.yml.example):

```yaml
name: Project Sync
on:
  issues:
    types:
      - opened
      - reopened
      - closed
jobs:
  sync:
    uses: SEU_USUARIO/.github/.github/workflows/project-sync.yml@master
    with:
      issue_url: ${{ github.event.issue.html_url }}
      issue_state: ${{ github.event.issue.state }}
      issue_action: ${{ github.event.action }}
      project_id: "PVT_kwXXXXXXXXXXXXXX"
      status_field: "PVTSSF_XXXXXXXXXXXXXXXXXXXXXXXXXX"
      status_realizar: "xxxxxxxx"
      status_em_andamento: "yyyyyyyy"
      status_realizado: "zzzzzzzz"
    secrets: inherit
```

**Linhas a editar:**

| Linha | Campo | O que colocar |
|---|---|---|
| 10 | `uses:` | Substituir `SEU_USUARIO` pelo seu usuário GitHub |
| 15 | `project_id:` | ID do projeto obtido no Passo 2 (`PVT_...`) |
| 16 | `status_field:` | ID do campo Status obtido no Passo 2 (`PVTSSF_...`) |
| 17 | `status_realizar:` | ID da opção "a fazer" obtido no Passo 2 |
| 18 | `status_em_andamento:` | ID da opção "em progresso" obtido no Passo 2 |
| 19 | `status_realizado:` | ID da opção "concluído" obtido no Passo 2 |

As linhas 12, 13 e 14 (`issue_url`, `issue_state`, `issue_action`) não precisam ser editadas — são preenchidas automaticamente pelo GitHub com os dados do evento.

---

## Observações importantes

**`github.event` não propaga em `workflow_call`**

Ao usar `workflow_call`, o contexto `github.event` não é passado automaticamente para o workflow chamado. Por isso o caller passa `issue_url`, `issue_state` e `issue_action` explicitamente via `with:`. Nunca tente acessar `github.event.issue.*` dentro de um workflow reutilizável — estará vazio.

**`PROJECT_TOKEN` deve existir no repo caller**

Secrets do repositório hospedeiro (este repo `.github`) não propagam para repositórios externos. O `PROJECT_TOKEN` precisa estar configurado no repositório que invoca o workflow ou ser herdado de conta/organização via `secrets: inherit`.

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

**Ordem correta ao publicar uma release**

Feche todos os issues relevantes primeiro, aguarde o workflow de changelog concluir e só então crie a release. Se criar a release antes de fechar os issues, eles aparecerão em uma seção "Unreleased" no changelog. Para corrigir: reabra e feche novamente o issue para retriggar o workflow.

---

## Estrutura deste repositório

```
.github/
  workflows/
    changelog.yml                      # workflow reutilizável — gera changelog.md
    project-sync.yml                   # workflow reutilizável — sincroniza issues com projeto
    run-changelog.yml                  # caller deste próprio repo
    changelog-caller.yml.example       # template para copiar para o seu repo
    project-sync-caller.yml.example    # template para copiar para o seu repo
README.md
changelog.md                           # gerado automaticamente
```
