# Padronização de Projeto (Papytron)

> Documento de padronização reaproveitável — MkDocs, GitHub Projects, política de Git
> (Git Flow rebase-only), labels, issues, PRs, pipeline de CI/CD e metodologia ágil
> (Scrum + XP). Pensado para reuso em trabalhos futuros.

---

## A. MkDocs — instruções para montar (passo a passo)

**1. Ambiente Python isolado** (na raiz do repo):
```bash
python -m venv .venv
source .venv/bin/activate
pip install mkdocs-material
pip freeze > docs-requirements.txt   # trava versões; o CI usa esse arquivo
```
Adicione `.venv/` e `site/` ao `.gitignore` (o `site/` é o build local).

**2. Estrutura inicial:**
```bash
mkdocs new .          # cria mkdocs.yml + docs/index.md
```
Você já tem `docs/` — só não deixe ele sobrescrever conteúdo seu.

**3. `mkdocs.yml` mínimo para solo** (cole e ajuste o nav):
```yaml
site_name: Papytron
site_url: https://<seu-usuario>.github.io/Papytron/
repo_url: https://github.com/<seu-usuario>/Papytron
repo_name: Papytron
plugins:
  - search
theme:
  name: material
  language: pt-BR
  features: [navigation.instant, navigation.tracking, navigation.tabs,
             navigation.sections, navigation.top, navigation.footer,
             search.suggest, search.highlight, content.code.copy]
  palette:
    - scheme: default
      toggle: {icon: material/weather-night, name: Modo escuro}
    - scheme: slate
      toggle: {icon: material/weather-sunny, name: Modo claro}
markdown_extensions:
  - admonition
  - attr_list
  - md_in_html
  - pymdownx.superfences:
      custom_fences:
        - {name: mermaid, class: mermaid, format: !!python/name:pymdownx.superfences.fence_code_format}
  - pymdownx.tabbed: {alternate_style: true}
nav:
  - Home: index.md
  - Arquitetura (MVCS-R): arquitetura/index.md
  - Backend (Spring): backend/index.md
  - Frontend (Angular): frontend/index.md
  - Banco (PostgreSQL): banco/index.md
  - Padrões do projeto:
    - Política de Git: padroes/git.md
    - Issues e PRs: padroes/issues-prs.md
    - Definição de Pronto/Feito: padroes/dod-dor.md
  - Testes (JUnit/Jasmine): testes/index.md
  - Sprints: sprints/index.md
  - Decisões (ADRs): adr/index.md
```

**4. Rodar local:** `mkdocs serve` → abre em `http://127.0.0.1:8000` com hot-reload.

**5. Deploy:** o workflow (`paths: docs/** + mkdocs.yml`) roda `mkdocs gh-deploy --force`,
que gera o HTML na branch `gh-pages` sozinho. Depois: **Settings → Pages → Source: branch
`gh-pages` / root**. Troque o `pip install mkdocs-material` do CI por
`pip install -r docs-requirements.txt` para builds reprodutíveis.

### Workflow de deploy do MkDocs (fonte na `main`)
```yaml
name: docs
on:
  push:
    branches: [main]
    paths: ['docs/**', 'mkdocs.yml']
permissions:
  contents: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: 3.x
      - run: pip install -r docs-requirements.txt
      - run: mkdocs gh-deploy --force
```

> **Branch da documentação:** para projeto **individual**, mantenha a fonte na `main`
> dentro de `docs/`. O `mkdocs gh-deploy` empurra o HTML para `gh-pages`
> automaticamente. Isolar a fonte numa branch `docs` (como o repo de equipe Parnas faz)
> é convenção de time — para solo é overhead. Só crie branch `docs` se for para *imitar
> o fluxo do time* como exercício didático.

> **Wiki:** dispensada. Não duplique documentação entre MkDocs e Wiki (divergem rápido).
> MkDocs é a fonte única da verdade.

---

## B. GitHub Projects — vale a pena? Sim.

Para Scrum solo/equipe pequena é a melhor peça gratuita do ecossistema, e **integra nativo
com Issues/PRs** (ao contrário da Wiki):

- Crie um **Project (board novo, tipo Table + Board)** no nível do repo ou do usuário.
- **Campos customizados** que valem ouro para Scrum:
  - `Status` (Backlog → Ready → In Progress → In Review → Done)
  - `Sprint` (campo do tipo *Iteration* — nativo, com datas e cadência automática!)
  - `Story Points` (number)
  - `Priority` (single select)
  - `Type` (US / Task / Bug / Spike)
- Use o campo **Iteration** para representar cada sprint (define duração, ex. 2 semanas, e
  ele cria as sprints futuras sozinho). Burndown sai do *Insights* do Project.
- **Automação:** Project → Workflows → ative "Item added to project → Status: Backlog",
  "PR merged → Status: Done", "Issue closed → Done". Zero manutenção manual.

Regra mental: **Project = onde o trabalho vive (planejamento/sprint). MkDocs = onde o
conhecimento estável vive (arquitetura/padrões/relatórios de sprint fechados).**

---

## C. Política de Git (Git Flow rebase-only)

### Modelo de branches
| Branch | Papel | Origem | Destino |
|---|---|---|---|
| `main` | produção, sempre estável e *taggeada* | — | recebe de `release/*` e `hotfix/*` |
| `develop` | integração contínua | `main` | recebe de `feature/*` |
| `feature/<id>-slug` | uma US/task | `develop` | volta p/ `develop` |
| `release/x.y.z` | estabilização de sprint | `develop` | vai p/ `main` (e reflui p/ `develop`) |
| `hotfix/x.y.z` | correção urgente em prod | `main` | vai p/ `main` (e reflui p/ `develop`) |

Nomeie a feature pelo número da issue: `feature/42-cadastro-usuario`.

### ⚠️ Correção conceitual importante (rebase-only)
"Não usar merge, só rebase" precisa de uma nuance, senão trava: **é impossível integrar
duas branches sem o comando merge**. O que você realmente quer é **história linear, sem
*merge commits***. Isso se consegue com **rebase + fast-forward**:

1. Você **rebase a feature *sobre* a develop** (reescreve os commits da feature em cima da
   ponta da develop).
2. Depois faz **`git merge --ff-only`** na develop — isso é o comando `merge`, mas como já
   está linear ele só *avança o ponteiro*, **sem criar commit de merge**.

Ou seja: o que some é o *merge commit*, não o comando. No GitHub, o botão equivalente é
**"Rebase and merge"**.

**Regra de ouro:** rebase só reescreve **branches que ainda são suas e privadas** (a
`feature/*` antes de virar oficial). **Nunca** rebase `develop`/`main` compartilhadas — por
isso "só com gente de confiança plena" faz sentido, mas a disciplina real é *só rebasear a
sua feature, nunca o tronco*.

### Fluxo prático de uma feature
```bash
git switch develop && git pull --ff-only
git switch -c feature/42-cadastro-usuario
# ... trabalha, commits livres ...

# antes de integrar: limpa a história
git rebase -i develop        # squash/reword/fixup -> commits limpos e atômicos
# resolve conflitos se houver, git rebase --continue

git switch develop && git pull --ff-only
git switch feature/42-cadastro-usuario
git rebase develop           # garante que está em cima da ponta atual

# integração linear, sem merge commit:
git switch develop
git merge --ff-only feature/42-cadastro-usuario
git push origin develop
git branch -d feature/42-cadastro-usuario
```
Se for via PR (recomendado p/ ter o CI como gate): abra PR `feature → develop`, deixe o CI
rodar, e use o botão **"Rebase and merge"**. Desabilite os outros botões (ver D/F).

### Release e Hotfix (o ponto delicado do rebase-only)
No Git Flow clássico, release/hotfix dão *back-merge* em dois lugares. Com modelo linear:
```bash
# Release
git switch -c release/1.2.0 develop
# só correções de bug + bump de versão aqui
# quando estável:
git switch main && git merge --ff-only release/1.2.0
git tag -a v1.2.0 -m "Release 1.2.0" && git push --tags
# refluxo p/ develop (history já é a mesma, então ff resolve):
git switch develop && git merge --ff-only main   # ou rebase develop onto main se divergiu
```
Para **hotfix**, como ele nasce da `main` e a `develop` já andou, o refluxo costuma
divergir — aí use **`git cherry-pick <hash-do-hotfix>`** na develop em vez de tentar ff.
Documente isso como a exceção aceita.

### Convenção de commits
Adote **Conventional Commits**: `tipo(escopo): descrição`
`feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `ci`, `perf`. Ex.:
`feat(backend): adiciona endpoint de cadastro de usuário`. Isso casa com o `rebase -i`
(você reescreve mensagens sujas em commits semânticos) e habilita changelog automático no
futuro.

---

## D. Labels (padrão reaproveitável)

Crie por categoria com prefixo (ajuda a filtrar):
- **Tipo:** `type: feature`, `type: bug`, `type: task`, `type: docs`, `type: spike`, `type: test`
- **Prioridade:** `prio: critical`, `prio: high`, `prio: medium`, `prio: low`
- **Status/fluxo:** `status: blocked`, `status: in-review`, `status: ready`
- **Camada (MVCS-R):** `layer: model`, `layer: view`, `layer: controller`, `layer: service`, `layer: repository`
- **Escopo:** `scope: backend`, `scope: frontend`, `scope: ci`, `scope: docs`
- **XP:** `xp: tech-debt`, `good first issue`, `xp: refactor`

Dica: dá pra versionar isso num `.github/labels.yml` e sincronizar com a action
`crazy-max/ghaction-github-labeler` — assim todo projeto futuro nasce com as mesmas labels.

---

## E. Padrão de Issues

Use **Issue Templates** em `.github/ISSUE_TEMPLATE/`. Dois mínimos:

**`user_story.md`** (Scrum):
```
Título: [US] Como <persona> quero <objetivo> para <benefício>
- Descrição / contexto
- Critérios de aceitação (checklist Given/When/Then)
- Story Points
- Definition of Ready atendida? (sim/não)
- Tarefas técnicas filhas (links)
```
**`task.md`** / **`bug.md`** (XP/manutenção): passos de reprodução, comportamento esperado
vs atual, ambiente, severidade.

Habilite **`config.yml`** com `blank_issues_enabled: false` para forçar o template.

---

## F. Padrão de Pull Requests

- **`.github/pull_request_template.md`:**
```
## O que
## Issue relacionada (Closes #)
## Como testar
## Checklist
- [ ] Commits seguem Conventional Commits e foram limpos via rebase -i
- [ ] Testes unitários e de integração passam localmente
- [ ] Documentação/ADR atualizada se necessário
- [ ] Branch rebaseada na ponta da develop
```
- **Settings → General → Pull Requests:** marque **só** "Allow rebase merging"; desmarque
  "merge commits" e "squash". Marque "Automatically delete head branches".
- **Branch protection (Settings → Branches)** para `main` e `develop`:
  - ✅ Require a pull request before merging
  - ✅ **Require linear history** (impede merge commit — casa com seu modelo)
  - ✅ Require status checks to pass → selecione os jobs do CI (ver G)
  - Em `main`, exija também o check de e2e/não-funcional.

---

## G. Pipeline de CI/CD (dois portões)

Dois workflows separados, refletindo exatamente os gates:

### `ci-develop.yml` — porta de entrada da `develop` (unit + integração)
```yaml
name: ci-develop
on:
  pull_request:
    branches: [develop]
jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: {distribution: temurin, java-version: '21', cache: maven}
      # unit (surefire) + integração (failsafe) num passo só:
      - run: mvn -B verify --file backend/pom.xml
  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: {node-version: '22', cache: npm, cache-dependency-path: frontend/package-lock.json}
      - run: npm ci --prefix frontend
      - run: npm test --prefix frontend -- --run   # vitest headless
```
> No Maven, separe os tipos: testes unitários terminam em `*Test` (surefire) e os de
> integração em `*IT` (failsafe). `mvn verify` roda os dois; `mvn test` só os unitários.
> Isso entrega o "unit + integração" pedido num comando.

### `ci-release.yml` — porta de entrada da `main` (e2e + não-funcionais)
```yaml
name: ci-release
on:
  pull_request:
    branches: [main]          # PRs de release/* ou hotfix/* p/ main
jobs:
  e2e:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: {POSTGRES_PASSWORD: test, POSTGRES_DB: papytron}
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready" --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      # sobe backend + frontend e roda Cypress/Playwright/Selenium
      # ... build backend, ng build, start, depois:
      - run: npx playwright test     # ou cypress run
  nao-funcionais:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: # carga/performance: k6 run load-test.js  (ou JMeter)
      - run: # acessibilidade/perf web: lighthouse-ci  (casa com o CLAUDE.md do front: AXE/WCAG AA)
      - run: # segurança: OWASP ZAP baseline scan (opcional)
```
Em **branch protection da `main`**, marque esses jobs como *required*. Assim, fisicamente,
**nada entra na `main` sem passar e2e + não-funcional**, e **nada entra na `develop` sem
unit + integração** — exatamente a regra desejada.

---

## H. Mapa Scrum + XP (como amarrar tudo)

- **Scrum:** sprints via campo *Iteration* do Project; backlog = Issues `type: feature`(US);
  planning estima Story Points; review fecha a sprint gerando um doc em
  `docs/sprints/sprint-0X.md` (consolidado); retro vira issues `xp: tech-debt`/`xp: refactor`.
- **XP:** CI obrigatório (integração contínua = os dois gates), *small releases* (branches
  `release/*` por sprint), *refactoring* contínuo (label própria + ADRs), *testing*
  (pirâmide unit→integração→e2e refletida nos workflows), *collective ownership*
  (rebase-only entre gente de confiança).
- **Definition of Ready / Definition of Done:** documente em `docs/padroes/dod-dor.md`.
  DoR = critérios de aceitação claros + estimada + sem bloqueio. DoD = código + testes
  passando nos gates + doc/ADR + PR rebaseado e aprovado.
- **ADRs** (Architecture Decision Records): uma pasta `docs/adr/` com um arquivo por
  decisão (ex.: "por que rebase-only", "por que MVCS-R"). Isso é o que torna o documento
  *reaproveitável* — você carrega as decisões padrão para o próximo trabalho.

---

## I. Como transformar isso em "kit reaproveitável"

Para futuros trabalhos, junte num **template repository** do GitHub contendo: `.github/`
(templates de issue/PR, `labels.yml`, os 3 workflows), `mkdocs.yml` base, `docs/padroes/*`
(git.md, issues-prs.md, dod-dor.md) e os ADRs. Marque o repo como **Template** (Settings →
Template repository). Aí cada novo projeto começa com "Use this template" e já nasce
padronizado.
