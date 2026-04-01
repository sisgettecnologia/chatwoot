# CI / GitHub Actions — paridade Chatwoot → Chatyn

Este guia serve para **reaplicar ou validar** o padrão Chatyn após **subir de versão** (ex.: `4.12.0` → `4.13.x`), quando o upstream adiciona ou altera `.github/workflows/**` e eventualmente `.circleci/config.yml`.

## Onde e quando corre (estado atual do repositório)

A **base do PR** é o ramo **destino** (ex.: PR de `feat/x` → `main` ⇒ base = `main`).

### GitHub Actions — validação / testes (CE)

| Workflow | Quando corre | Bases de PR permitidas (`pull_request.branches`) | O que executa |
|----------|--------------|--------------------------------------------------|---------------|
| [`run_foss_spec.yml`](../../.github/workflows/run_foss_spec.yml) | `pull_request`, `workflow_dispatch` | `develop`, `main`, `release/chatyn-v*` | Rubocop; ESLint; `pnpm test:coverage`; RSpec CE em 16 jobs (PG16, Redis), strip `enterprise` |
| [`frontend-fe.yml`](../../.github/workflows/frontend-fe.yml) | `pull_request` | `develop`, `main`, `release/chatyn-v*` | ESLint + `test:coverage` |
| [`test_docker_build.yml`](../../.github/workflows/test_docker_build.yml) | `pull_request`, `workflow_dispatch` | `develop`, `main`, `release/chatyn-v*` | Build `docker/Dockerfile` amd64 + arm64, sem push |
| [`size-limit.yml`](../../.github/workflows/size-limit.yml) | `pull_request` | `develop`, `main`, `release/chatyn-v*` | `assets:precompile` + `pnpm run size` |
| [`logging_percentage_check.yml`](../../.github/workflows/logging_percentage_check.yml) | `pull_request` | `develop`, `main`, `release/chatyn-v*` | % mínimo de linhas `Rails.logger` em `.rb` alterados |
| [`run_mfa_spec.yml`](../../.github/workflows/run_mfa_spec.yml) | `pull_request` | `develop`, `main`, `release/chatyn-v*` | RSpec MFA (com `if:` repositório + bloqueio Dependabot); o job referencia `workflow_dispatch` no `if` mas **`on:` não declara** `workflow_dispatch` |
| [`lint_pr.yml`](../../.github/workflows/lint_pr.yml) | `pull_request_target` | *qualquer base* | Título do PR (convencional) |
| [`auto-assign-pr.yml`](../../.github/workflows/auto-assign-pr.yml) | `pull_request` (`opened`) | *qualquer base* | Atribui PR ao autor |

### GitHub Actions — publicação / housekeeping (não são “suíte de testes” do PR)

| Workflow | Quando corre | Notas |
|----------|--------------|--------|
| [`publish_foss_docker.yml`](../../.github/workflows/publish_foss_docker.yml) | `push` **só** tags `v*`, `workflow_dispatch` | Build multi-arch CE, digest, manifest, push Docker Hub (`sisgettecnologia/chatyn`) |
| [`publish_codespace_image.yml`](../../.github/workflows/publish_codespace_image.yml) | `workflow_dispatch` | GHCR Codespace |
| [`stale.yml`](../../.github/workflows/stale.yml) | `schedule` (cron) | PRs stale |
| [`lock.yml`](../../.github/workflows/lock.yml) | `schedule`, `workflow_dispatch` | Lock de threads (gate `sisgettecnologia/chatyn`) |
| [`nightly_installer.yml`](../../.github/workflows/nightly_installer.yml) | `schedule`, `workflow_dispatch` | Script instalador Linux upstream |

**EE:** [`publish_ee_docker.yml`](../../.github/workflows/publish_ee_docker.yml) — **fora do escopo** deste guia; não entra na tabela CE.

### CircleCI ([`.circleci/config.yml`](../../.circleci/config.yml))

Workflow **`build`**: **lint** (swagger, bundle audit, Rubocop, ESLint) → **frontend-tests** (`pnpm test:coverage`) → **backend-tests** (18×, OpenSearch, PG16, RSpec com coverage) → **coverage** (Qlty + artefactos) → **build** (job vazio compat. com checks).

**Quando corre:** o YAML **não** define filtros por branch; o disparo efetivo controla-se no **painel do CircleCI**. No Chatyn, o pipeline ligado via **GitHub App** está configurado para eventos de **PR** (por exemplo *Non-draft PR opened*, *PR marked ready for review*, *Pushes to open non-draft PRs*). O pipeline **OAuth legado** está **sem** trigger automático. Se mudarem integrações ou triggers, atualizar este parágrafo.

---

## Política em `.github/workflows` (resumo CE)

| Regra | Detalhe (espelha o repo hoje) |
|-------|-------------------------------|
| **PR** | A suíte CE **principal** de validação corre em `pull_request` com bases **`develop`**, **`main`** e **`release/chatyn-v*`** (os seis workflows da primeira tabela). Workflows **auxiliares** como **`lint_pr`** e **`auto-assign-pr`** usam **escopo mais alargado** (*qualquer* base). |
| **`push` em branch** | **Não** há workflows CE de **teste** em `on.push.branches`; validação em PR ou manual (`workflow_dispatch`) onde aplicável |
| **Tags** | `publish_foss_docker.yml` dispara em **`push` de tag `v*`** (é `push`, não workflow de teste) |
| **Publish CE** | Tags `v*` + **`workflow_dispatch`** |

**Divergência intencional vs Chatwoot:** menos dependência de `push` em branches para integração; publicação CE alinhada a **tag `v*`** ou **dispatch** manual; suíte pesada concentrada em **PR** (e CircleCI, conforme configuração).

**Justificativa operacional:** reduzir minutos em commits diretos em `feat/*` / `fix/*`; a barreira de merge é o PR com a suíte completa.

**Histórico de referência no Chatyn**

- `a83acab5878373f08e3d50e5ff8eb1ef09c95643` — `ci(docker): ajusta publish CE do Chatyn (registry e branch main)` (contexto anterior de `publish_foss_docker.yml`).
- Ajustes posteriores: ramos, slugs de repositório, remoção do `deploy_check` Heroku-specific, correção do `pnpm/action-setup` em `run_foss_spec`.

---

## CircleCI

O ficheiro [`.circleci/config.yml`](../../.circleci/config.yml) define o workflow `build` (**lint**, **frontend-tests**, **backend-tests**, **coverage**, **build**) **sem** filtros de branch no YAML: **não** há workaround de filtros no repositório; o que decide *quando* o pipeline corre é o **painel** e a integração GitHub.

**Estado atual (Chatyn):** o pipeline **GitHub App** dispara em eventos de **pull request** (nomeadamente *Non-draft PR opened*, *PR marked ready for review*, *Pushes to open non-draft PRs*). A ligação **OAuth legada** permanece **sem** triggers automáticos. Em catch-up com upstream, continuar a fazer merge de alterações ao `config.yml` quando necessário; se alterarem triggers ou integrações no CircleCI, atualizar também a secção **Onde e quando corre** / este parágrafo para o documento não ficar desatualizado.

---

## Regras de mapeamento (resumo — conteúdo / fork)

| Tema | Chatwoot (typical `develop`) | Chatyn |
|------|------------------------------|--------|
| Branch de produção no mapa CE | `master` em gatilhos / condições shell | **`main`** (substitui `master`, **não** manter os dois na mesma lista onde customizámos) |
| **Gatilhos CE (FOSS spec, frontend-fe, test docker, size, logs, MFA)** | mistura de `push` / `pull_request` / tags | Suíte principal em `pull_request` com bases **`develop` + `main` + `release/chatyn-v*`**; auxiliares (`lint_pr`, `auto-assign-pr`) em escopo mais alargado |
| **Publish CE** | `push` + branches/tags típico | **`push` apenas `tags: v*`** + **`workflow_dispatch`** |
| Repositório no `if:` / gates | `chatwoot/chatwoot` | **`sisgettecnologia/chatyn`** (ex.: `run_mfa_spec`, `lock`) |
| Imagem CE Docker Hub | (N/A no teu fork) | `DOCKER_REPO: sisgettecnologia/chatyn`; **`ref_name`** no workflow distingue `main` (**`latest-ce`**) vs outro ref (ex. tag **`v*`-ce**) |
| GHCR Codespace (manual) | `ghcr.io/chatwoot/chatwoot_codespace:latest` | `ghcr.io/sisgettecnologia/chatyn_codespace:latest` |
| `deploy_check` (Heroku review app) | `pull_request` + `curl` a `*.herokuapp.com` | **Removido** no Chatyn — não há review app equivalente |
| **EE** (`publish_ee_docker.yml` etc.) | manter como upstream | **Fora do escopo** deste guia de gatilhos CE |

---

## Checklist após merge/catch-up com upstream

1. **Listar o que mudou no upstream**
   - `git diff <tag-chatyn-anterior>..<tag-upstream-nova> -- .github/workflows/ .circleci/`
   - Ou: comparar pasta `.github/workflows` contra `https://github.com/chatwoot/chatwoot/tree/develop/.github/workflows`.

2. **Novos ficheiros `*.yml`**
   - Para cada workflow novo: decidir se corre no Chatyn, se precisa de **slug** `sisgettecnologia/chatyn`, e aplicar **`master` → `main`** onde fizer sentido no CE.

3. **Ficheiros que já customizámos (revalidar sempre)**
   - `publish_foss_docker.yml` — **`on.push.tags: v*`** e **`workflow_dispatch`** (sem `on.push.branches`); `DOCKER_REPO` Chatyn; validação de build no PR em **`test_docker_build.yml`**.
   - `run_foss_spec.yml` — **sem** `push`; `pull_request` com bases **`develop` + `main` + `release/chatyn-v*`** (confirmar se queres manter `develop`); **não** colocar `ref`/`repository` no **`pnpm/action-setup`** (só em `actions/checkout` se necessário).
   - `run_mfa_spec.yml` — `github.repository == 'sisgettecnologia/chatyn'`; bases **`develop` + `main` + `release/chatyn-v*`**; opcional alinhar `on:` com `workflow_dispatch` se quiseres reruns manuais.
   - `lock.yml` — mesmo gate de repositório.
   - `publish_codespace_image.yml` — tag GHCR Chatyn.
   - `frontend-fe.yml`, `size-limit.yml`, `logging_percentage_check.yml` — bases **`develop` + `main` + `release/chatyn-v*`** (alinhado com **`test_docker_build.yml`** nestes três).
   - Se o upstream **reintroduzir** `deploy_check.yml`: **não** adoptar como está; ou validar URL/infra real ou manter removido / substituir por no-op manual.

4. **CircleCI**
   - Fazer merge de alterações upstream ao `config.yml` quando necessário; triggers e GitHub App vs OAuth conforme secção **CircleCI** (hoje: App em eventos de PR, OAuth sem trigger automático).

5. **Smoke mental**
   - PR **para `main`**: corre **`run_foss_spec`**, **`frontend-fe`**, **`test_docker_build`**, **`size-limit`**, **`logging_percentage_check`**, **`run_mfa_spec`** (e **`lint_pr` / auto-assign** conforme aplicável).
   - PR **para `develop`**: corre os workflows que listam `develop` na base, incluindo **`run_mfa_spec`**.
   - PR **para `release/chatyn-v*`**: em geral a maior parte da suíte corre; comparar com a tabela **Onde e quando corre**.
   - **Push em branch** (sem tag): não há workflows CE de **teste** em `on.push.branches`.
   - Imagem CE: tag **`v*`** ou **Publish Chatyn CE docker images** (dispatch).
   - Nada a exigir `Deploy Check` Heroku nos branch protections.

---

## Onde **não** procurar 1:1 de ficheiro

- **EE** e registry `chatwoot/chatwoot` — fora do escopo de gatilhos CE aqui.
- **Heroku** — só relevante se um dia existir preview URL próprio e um workflow novo que faça probe dessa URL.

---

## Manutenção deste documento

Ao fechar um porte grande (ex. `4.12` → `4.13`), vale acrescentar uma linha na secção **Histórico** com tag/branch e nota curta (“upstream adicionou workflow X — aplicado Y no Chatyn”). Se a **política de gatilhos** mudar, atualizar primeiro a secção **Política em `.github/workflows` (resumo CE)** e o checklist.
