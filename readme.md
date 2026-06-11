# mauriciogofas/.github — Workflows Globais

Repositório central de workflows reutilizáveis para todos os repositórios da conta [`mauriciogofas`](https://github.com/mauriciogofas).

Dois workflows estão disponíveis:

1. **`changelog.yml`** — gera e commita `changelog.md` automaticamente a cada release publicada ou issue fechada
2. **`project-sync.yml`** — adiciona issues ao GitHub Projects V2 e move entre colunas conforme o status

---

## Como funciona

### `changelog.yml`

A cada release publicada ou issue fechada/editada, o workflow reconstrói o `changelog.md` **completo do zero** a partir de todos os dados do repositório — sem depender do conteúdo anterior do arquivo. Isso significa que apagar o `changelog.md` e fechar qualquer issue o regenera integralmente com todo o histórico.

**Estrutura gerada:**

```
# Changelog

## [Próxima atualização](url)

**Melhorias:**
- Título do issue | #N `label`

**Correções:**
- Título do issue | #N `label`

**Docs:**              ← seção extra quando 2+ issues têm a mesma label não-padrão
- ...

**Vale mencionar:**    ← issues com labels avulsas (sem grupo suficiente)
- ...

[Comparar versões](url)

## [v2.0.0 - 10/06/2026](url)
...
```

**Classificação automática:**

| Seção | Critério |
|---|---|
| **Melhorias** | Label `enhancement`, `melhoria`, `melhorias` — ou sem label alguma |
| **Correções** | Label contém: `bug`, `erro`, `error`, `fix`, `hotfix` |
| **Seção própria** | 2+ issues com a mesma label não-padrão |
| **Vale mencionar** | Issues com label não-padrão sem grupo suficiente |

A classificação verifica Melhorias primeiro: se o issue tem label `melhoria` e também `bug`, vai para Melhorias. Correções só é aplicado quando não há label de melhoria.

- Issues com múltiplas labels são classificados pela label de maior prioridade (Correções > Melhorias > demais)
- Labels são exibidas à direita do issue com cor `#a99c9c`, sem sublinhado e com link para filtro no GitHub
- Nomes de seções extras são capitalizados automaticamente (`docs` → `Docs:`)
- O bloco "Próxima atualização" lista issues fechados após a última release
- Cada bloco de release lista apenas os issues fechados no intervalo entre ela e a release anterior

**Quando uma release é criada:**

O bloco "Próxima atualização" é automaticamente convertido em `## [v2.0.0 - 10/06/2026]` com link para a release e "Comparar versões" apontando para o diff com a versão anterior.

---

### `project-sync.yml`

A cada issue aberta, reaberta ou fechada, o workflow:

1. Resolve o `node_id` da issue via GraphQL
2. Adiciona ao projeto via `addProjectV2ItemById`
3. Atualiza o campo **Status** conforme a ação:
   - `opened` → **Realizar**
   - `reopened` → **Em andamento**
   - `closed` → **Realizado**
4. Move o item para o topo do board via `updateProjectV2ItemPosition`

---

## Configuração

### Passo 1 — Criar o projeto no GitHub

Acesse **Projects > New project > Board** e crie três colunas:

- **Realizar** — issues a fazer
- **Em andamento** — issues em progresso
- **Realizado** — issues concluídas

### Passo 2 — Obter os IDs do projeto e das colunas

```bash
# Listar projetos e obter o ID (formato PVT_...)
gh api graphql -f query='
query {
  user(login: "SEU_USUARIO") {
    projectsV2(first: 10) {
      nodes { number title id }
    }
  }
}'
```

Com o ID do projeto, obter os IDs do campo Status e das colunas:

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

A resposta retorna:
- `id` do campo Status → formato `PVTSSF_...` → valor de `status_field`
- `id` de cada opção → formato hexadecimal curto → valores de `status_realizar`, `status_em_andamento`, `status_realizado`

### Passo 3 — Configurar o secret `PROJECT_TOKEN`

Crie um Personal Access Token com escopos `repo` e `project` em **Settings > Developer settings > Personal access tokens**.

Adicione como secret no repositório:
**Settings > Secrets and variables > Actions > New repository secret**
Nome: `PROJECT_TOKEN`

> **Como funciona o `secrets: inherit`:** quando seu repositório usa `secrets: inherit` para chamar um workflow em `mauriciogofas/.github`, o GitHub passa automaticamente os secrets do seu repositório para o workflow — desde que seu repositório também pertença à conta `mauriciogofas`. Se o seu repositório estiver em outra conta ou organização, o `secrets: inherit` não funciona cross-owner: o workflow chamado recebe os secrets vazios. Nesse caso, faça um fork de `mauriciogofas/.github` para a sua própria conta e aponte o caller para o seu fork — assim os secrets propagam normalmente.

### Passo 4 — Criar os callers no repositório

Copie os arquivos [`.example`](../../tree/master/.github/workflows) como ponto de partida. Estão em [`.github/workflows/`](../../tree/master/.github/workflows) e não são executados pelo GitHub Actions por terem extensão `.example`.

**`.github/workflows/changelog-caller.yml`**

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
    uses: mauriciogofas/.github/.github/workflows/changelog.yml@master
    with:
      git_user: "seu-usuario-github"
      git_name: "Seu Nome"
      git_email: "seu@email.com"
    secrets: inherit
```

| Linha | Campo | O que colocar |
|---|---|---|
| 11 | `uses:` | Manter `mauriciogofas` se seu repo pertence à conta `mauriciogofas`. Substituir pelo seu usuário **somente se tiver feito fork** do `.github` para outra conta |
| 13 | `git_user:` | Seu usuário GitHub (usado na URL do push) |
| 14 | `git_name:` | Seu nome para o commit |
| 15 | `git_email:` | Seu email para o commit |

**`.github/workflows/project-sync-caller.yml`**

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
    uses: mauriciogofas/.github/.github/workflows/project-sync.yml@master
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

| Linha | Campo | O que colocar |
|---|---|---|
| 10 | `uses:` | Manter `mauriciogofas` se seu repo pertence à conta `mauriciogofas`. Substituir pelo seu usuário **somente se tiver feito fork** do `.github` para outra conta |
| 15 | `project_id:` | ID do projeto — Passo 2 |
| 16 | `status_field:` | ID do campo Status — Passo 2 |
| 17 | `status_realizar:` | ID da opção "a fazer" — Passo 2 |
| 18 | `status_em_andamento:` | ID da opção "em progresso" — Passo 2 |
| 19 | `status_realizado:` | ID da opção "concluído" — Passo 2 |

As linhas 12, 13 e 14 (`issue_url`, `issue_state`, `issue_action`) são preenchidas automaticamente pelo GitHub — não editar.

---

## Observações importantes

**Issues excluídos saem do projeto automaticamente**

Quando um issue é excluído no GitHub, ele é removido do GitHub Projects automaticamente — esse comportamento é nativo da plataforma e independe deste workflow.

**Fechar e reabrir um issue**

Além de mover o issue para a coluna **Em andamento** no projeto, reabrir um issue faz com que ele reapareça na seção **Próxima atualização** do changelog na próxima execução — já que volta a ser um issue fechado após a última release. É um comportamento natural do fluxo que funciona como indicação de retrabalho em andamento.

**`github.event` não propaga em `workflow_call`**

O contexto `github.event` não é passado automaticamente para o workflow chamado. Por isso o caller passa `issue_url`, `issue_state` e `issue_action` explicitamente via `with:`. Nunca tente acessar `github.event.issue.*` dentro de um workflow reutilizável — estará vazio.

**Ordem correta ao publicar uma release**

Feche todos os issues relevantes primeiro e aguarde o workflow de changelog concluir. Criar a release antes de fechar os issues faz com que eles apareçam no próximo ciclo em vez do bloco da release atual.

**`section_map` opcional**

O `changelog.yml` aceita um input opcional `section_map` (JSON) para sobrescrever os títulos das seções. Exemplo no caller:

```yaml
with:
  section_map: '{"bug":"Bugs corrigidos","enhancement":"Funcionalidades"}'
```

---

## Estrutura do repositório

```
.github/
  workflows/
    changelog.yml                      # workflow reutilizável — gera changelog.md
    project-sync.yml                   # workflow reutilizável — sincroniza issues com projeto
    run-changelog.yml                  # caller deste próprio repo
    changelog-caller.yml.example       # template para copiar para o seu repo
    project-sync-caller.yml.example    # template para copiar para o seu repo
readme.md
changelog.md                           # gerado automaticamente
```

---

## Nota técnica sobre privacidade

Os workflows deste repositório não coletam, armazenam nem recebem nenhum dado sobre quem os utiliza. Quando um repositório externo chama `uses: mauriciogofas/.github/...`, o workflow roda inteiramente no runner do GitHub associado ao repositório caller — não neste. Runs de workflows reutilizáveis chamados externamente aparecem apenas no repositório caller e são invisíveis para `mauriciogofas/.github`. Para que fosse possível receber qualquer dado, o workflow precisaria fazer uma chamada explícita para um endpoint externo — o que não existe aqui. O código dos workflows pode ser auditado diretamente neste repositório.

---

<p align="center">© 2026 <a href="https://gofas.net/?utm_source=github&utm_medium=readme&utm_campaign=opensource">Gofas Software</a></p>
