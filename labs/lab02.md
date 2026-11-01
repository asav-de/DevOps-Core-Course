# Lab 2 — Docker Containerization

Лаба 2 — контейнеризация в Docker

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Containerization-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Docker%2029-informational)

> **Goal:** Containerize your Lab 1 service with a hand-written, multi-stage Dockerfile; scan the image; push it to a registry. By the end of this week you have *the* artifact that every later lab deploys.
> **Deliverable:** A PR from `lab02` adding `app_python/Dockerfile`, `.dockerignore`, a `docs/LAB02.md` report, and (optionally) the Go distroless bonus under `app_go/`.
>
> **Цель:** упаковать сервис из лабы 1 в контейнер с написанным вручную многоступенчатым Dockerfile; просканировать образ и запушить его в реестр. К концу недели у тебя будет *тот самый* артефакт, который деплоят все последующие лабы.
> **Что сдать:** PR из ветки `lab02`, добавляющий `app_python/Dockerfile`, `.dockerignore`, отчёт `docs/LAB02.md` и (по желанию) бонус с distroless на Go в `app_go/`.

---

## Overview

Обзор

In Lab 1 your service "worked on your machine." This lab makes it **work everywhere**. The actual skill being graded is *writing a Dockerfile line-by-line*: which base image, in which order, as which user, with which entrypoint. The skeletons below show the **directives** but not the **values** — you fill those in and you defend the choice in `docs/LAB02.md`.

В лабе 1 твой сервис «работал на твоей машине». Эта лаба делает так, чтобы он **работал везде**. На самом деле здесь оценивают навык *писать Dockerfile строка за строкой*: какой базовый образ взять, в каком порядке идти, от какого пользователя запускать и с каким entrypoint. Скелеты ниже показывают **директивы**, но не **значения** — их ты подставляешь сам и обосновываешь выбор в `docs/LAB02.md`.

In this lab you will practice:

В этой лабе ты потренируешься:

- Writing production-ready, layer-cache-friendly Dockerfiles by hand
  * Писать вручную Dockerfile'ы, готовые к продакшену и дружелюбные к кешу слоёв
- Running containers as a non-root user (defense-in-depth against container escape)
  * Запускать контейнеры от непривилегированного пользователя (защита в глубину от выхода из контейнера)
- Multi-stage builds for small, reproducible images
  * Собирать многоступенчатые образы — маленькие и воспроизводимые
- Image vulnerability scanning with Trivy and CI-style exit-code gating
  * Сканировать образы на уязвимости с Trivy и настраивать CI-проверку по коду возврата
- Tagging by SemVer/SHA and pushing to a public registry
  * Тегировать по SemVer/SHA и пушить в публичный реестр

> ⚠️ **Scope:** Compose, Kubernetes, and CI gates come later. This lab is one service, one image, one registry.
>
> ⚠️ **Границы:** Compose, Kubernetes и CI-проверки будут позже. В этой лабе — один сервис, один образ, один реестр.

**Tech stack (May 2026):** Docker Engine 29.x recommended (containerd image store default); **23+ also works** — BuildKit was on by default in 23 and `# syntax=docker/dockerfile:1` is honoured · `python:3.13-slim` · Trivy v0.69.3+

**Стек (май 2026):** рекомендуется Docker Engine 29.x (по умолчанию используется containerd image store); **подойдёт и 23+** — в 23-й BuildKit уже включён по умолчанию, а директива `# syntax=docker/dockerfile:1` соблюдается · `python:3.13-slim` · Trivy v0.69.3+

---

## Project State

Состояние проекта

**You should have from previous labs:**

**У тебя уже должно быть из предыдущих лаб:**

- `app_python/` — the Flask/FastAPI/Django service from Lab 1 with `GET /` and `GET /health`
  * `app_python/` — сервис на Flask/FastAPI/Django из лабы 1 с `GET /` и `GET /health`
- *(optional)* `app_go/` (or other compiled-language sibling) from the Lab 1 bonus
  * *(опционально)* `app_go/` (или другой сервис на компилируемом языке) из бонуса лабы 1

**This lab adds:**

**Эта лаба добавляет:**

- `app_python/Dockerfile` — single-stage, then refactored to multi-stage
  * `app_python/Dockerfile` — сначала одноступенчатый, затем переписанный в многоступенчатый
- `app_python/.dockerignore`
  * `app_python/.dockerignore` — файл исключений для контекста сборки
- `app_python/docs/LAB02.md` — your submission report
  * `app_python/docs/LAB02.md` — твой отчёт по сдаче
- A published image at `ghcr.io/<you>/devops-info-service:1.0.0` (or Docker Hub equivalent)
  * Опубликованный образ `ghcr.io/<you>/devops-info-service:1.0.0` (или аналог в Docker Hub)
- *(bonus)* `app_go/Dockerfile` — multi-stage with a distroless final stage
  * *(бонус)* `app_go/Dockerfile` — многоступенчатый, с финальной distroless-стадией

By Lab 9 Kubernetes pulls this image; by Lab 13 ArgoCD ships new versions of it. The tag you push today is the artifact reference for the next 14 weeks.

К лабе 9 этот образ будет тянуть Kubernetes, а к лабе 13 ArgoCD начнёт поставлять его новые версии. Тег, который ты запушишь сегодня, станет ссылкой на артефакт на следующие 14 недель.

---

## Setup

Настройка

```bash
docker version              # Engine 29.x recommended; 23+ works (BuildKit on by default since 23); rootless OK
docker buildx version       # BuildKit is required (bundled)
trivy --version             # v0.69.3 or newer — NOT v0.69.4 (see warning below)
```

Пояснение: первая команда проверяет версию Docker Engine, вторая — наличие BuildKit (buildx), третья — версию Trivy. Версия Trivy критична: v0.69.4 скомпрометирована, используй v0.69.3 или более поздний чистый патч.

> ⚠️ **Trivy supply-chain warning.** Install **v0.69.3** or a later clean patch. **Never install `v0.69.4`** — that release was compromised (CVE-2026-33634, the March 2026 supply-chain attack). If you wire Trivy into GitHub Actions in Lab 3, pin `trivy-action@v0.35.0+` / `setup-trivy@v0.2.6+`.
>
> ⚠️ **Предупреждение о цепочке поставок Trivy.** Ставь **v0.69.3** или более поздний чистый патч. **Никогда не устанавливай `v0.69.4`** — этот релиз был скомпрометирован (CVE-2026-33634, атака на цепочку поставок в марте 2026). Если подключаешь Trivy в GitHub Actions в лабе 3, закрепляй `trivy-action@v0.35.0+` / `setup-trivy@v0.2.6+`.

> 🌐 **Trivy DB mirror.** On rate-limited or restricted networks the default DB pull from `ghcr.io/aquasecurity/trivy-db` returns `DENIED`. If that happens, pass `--db-repository public.ecr.aws/aquasecurity/trivy-db:2` to every `trivy image` call. This is the one operational gotcha worth remembering — your campus network might bite.
>
> 🌐 **Зеркало базы Trivy.** В сетях с ограничениями или rate-limit'ами загрузка базы по умолчанию из `ghcr.io/aquasecurity/trivy-db` возвращает `DENIED`. В таком случае передавай `--db-repository public.ecr.aws/aquasecurity/trivy-db:2` в каждый вызов `trivy image`. Это тот самый эксплуатационный подводный камень, о котором стоит помнить — университетская сеть вполне может его преподнести.

Create the layout (you'll write the contents yourself):

Создай структуру каталогов (содержимое файлов ты напишешь сам):

```
app_python/
├── Dockerfile             # YOU WRITE — Task 1 → refactor in Task 3
├── .dockerignore          # YOU WRITE — Task 1
├── README.md              # extend with a Docker section in Task 5
├── docs/
│   └── LAB02.md           # YOU WRITE — submission report
└── (your Lab 1 sources)
```

Пояснение: в каталоге `app_python/` должны появиться `Dockerfile`, `.dockerignore`, расширенный `README.md` и отчёт `docs/LAB02.md`; исходники из лабы 1 остаются на месте.

---

## Task 1 — Write a Production Dockerfile (3 pts)

Задание 1 — Напиши production-Dockerfile (3 балла)

### 1.1 — The single-stage Dockerfile

1.1 — Одноступенчатый Dockerfile

`YOUR TASK`: write `app_python/Dockerfile` from the skeleton below. Every `___` is yours. There is **no fully-formed example** in this lab — by the time a Dockerfile is given to you whole, it's not learning, it's typing practice. The lecture (slides 11, 13, 14, 15) explains the *why* for every line; bring it.

`YOUR TASK`: напиши `app_python/Dockerfile` по скелету ниже. Каждый `___` — твой. **Полностью готового примера** в этой лабе нет — если Dockerfile дают целиком, это уже не обучение, а тренировка набора текста. Лекция (слайды 11, 13, 14, 15) объясняет *почему* в каждой строке; принеси эти знания с собой.

```dockerfile
# syntax=docker/dockerfile:1
FROM ___                  # YOUR TASK: pin a slim Python base. Why slim? Why pinned? (slide 15)
WORKDIR ___               # YOUR TASK: absolute path. Not / and not ~
RUN ___                   # YOUR TASK: create a NUMERIC non-root user (uid >= 10000)
COPY ___ ___              # YOUR TASK: one file FIRST — the one that changes rarely
RUN pip install --___ -r ___  # YOUR TASK: which flag stops pip from baking its cache into the layer?
COPY ___ ___              # YOUR TASK: app source LAST. Explain the layer-cache reasoning in docs
USER ___                  # YOUR TASK: switch to the numeric UID from above
EXPOSE ___                # YOUR TASK: which port does your Lab 1 app listen on?
# YOUR TASK (recommended): HEALTHCHECK pinging GET /health. slim has NO curl + NO wget — pick deliberately
CMD [___]                 # YOUR TASK: exec form, not shell form (PID 1 trap → SIGTERM lost)
```

Пояснение: каждая строка с `___` — задание на выбор значения. Порядок директив важен: сначала зависимости (`requirements.txt`), потом исходники, — так слой с зависимостями переиспользуется между сборками. `CMD` записывается в exec-форме (JSON-массив), чтобы приложение стало PID 1 и получало SIGTERM.

**Each blank is a rubric line.** A "good" answer matches the lecture's reasoning; a "lucky" answer that compiles but can't be defended in `docs/LAB02.md` will lose points.

**Каждый пропуск — строка рубрики.** «Хороший» ответ совпадает с логикой лекции; «удачный» ответ, который собирается, но не обоснован в `docs/LAB02.md`, потеряет баллы.

### 1.2 — `.dockerignore`

1.2 — Файл `.dockerignore`

`YOUR TASK`: write `app_python/.dockerignore` so `COPY . .` doesn't ship `.git/`, virtualenvs, IDE configs, `__pycache__/`, or `.env` into the image.

`YOUR TASK`: напиши `app_python/.dockerignore`, чтобы `COPY . .` не отправлял в образ `.git/`, виртуальные окружения, конфиги IDE, `__pycache__/` и `.env`.

```gitignore
# YOUR TASK: list at minimum
# - VCS metadata (the Toyota 2022 .git leak — slide 16)
# - Python caches (__pycache__, *.pyc)
# - Virtualenvs (.venv, venv/)
# - Local env files (.env)
# - Editor dirs (.idea/, .vscode/)
# - Docs/tests (not needed at runtime — smaller context, faster builds)
```

Пояснение: перечисли минимум эти категории — метаданные VCS, кеши Python, виртуальные окружения, локальные `.env`-файлы, каталоги редакторов и документацию с тестами, которые не нужны в рантайме.

Hint: same syntax as `.gitignore`, but evaluated by the Docker build context — *not* git. A `.dockerignore` is mandatory for this task; missing it costs the entire `.dockerignore` rubric line.

Подсказка: синтаксис тот же, что у `.gitignore`, но файл учитывается контекстом сборки Docker, а *не* git. `.dockerignore` для этого задания обязателен; без него ты теряешь всю строку рубрики про `.dockerignore`.

### 1.3 — Proof of work

1.3 — Подтверждение работы

**Paste into `docs/LAB02.md`:**

**Вставь в `docs/LAB02.md`:**

- The contents of your `Dockerfile` (yes, the whole file — it's short and it's the artifact being graded)
  * Содержимое твоего `Dockerfile` (да, целиком — он короткий и именно его оценивают)
- The contents of your `.dockerignore`
  * Содержимое твоего `.dockerignore`
- A one-sentence justification per `# YOUR TASK` line — *why this value*, not *what it is*
  * Обоснование в одном предложении на каждую строку `# YOUR TASK` — *почему именно это значение*, а не *что это такое*
- Output of `docker build -t devops-info-service:lab02 ./app_python` showing the final image hash (illustrative — your numbers will differ)
  * Вывод `docker build -t devops-info-service:lab02 ./app_python` с финальным хешем образа (пример — твои числа будут другими)

<details>
<summary>💡 Why each rule matters (read before you write)</summary>

💡 Зачем нужно каждое правило (прочитай перед тем, как писать)

- **Pinned slim base** — `python:3.13` ≈ 1.2 GB, `python:3.13-slim` ≈ 150 MB. Smaller image = faster pulls (Lab 9 K8s autoscale), fewer packages = fewer CVEs (Task 4). `:latest` is mutable so two builds from "the same Dockerfile" can produce different images.
  * **Зафиксированный slim-образ** — `python:3.13` ≈ 1,2 ГБ, `python:3.13-slim` ≈ 150 МБ. Меньше образ — быстрее загрузка (автомасштабирование K8s в лабе 9), меньше пакетов — меньше CVE (задание 4). Тег `:latest` изменяем, поэтому две сборки «из одного и того же Dockerfile» могут дать разные образы.
- **Numeric non-root UID** — A kernel container-escape CVE (e.g. CVE-2022-0185) gives root on the host **if your process is root**. `USER 10001` neutralises it. Numeric (vs `USER appuser`) keeps working under `runAsNonRoot: true` admission policies you'll meet in Lab 9+.
  * **Числовой UID непривилегированного пользователя** — уязвимость ядра для выхода из контейнера (например, CVE-2022-0185) даёт root на хосте, **если твой процесс запущен от root**. `USER 10001` это нейтрализует. Числовой UID (в отличие от `USER appuser`) продолжает работать при политиках доступа `runAsNonRoot: true`, с которыми ты столкнёшься в лабе 9+.
- **Layer order (deps before code)** — Each instruction is a cached layer. `COPY . . && pip install` re-installs every dependency on every 1-character code change. `COPY requirements.txt . && pip install && COPY . .` re-uses the dep layer until `requirements.txt` actually changes.
  * **Порядок слоёв (зависимости до кода)** — каждая инструкция — это кешируемый слой. При `COPY . . && pip install` все зависимости переустанавливаются при любом изменении кода даже на один символ. Схема `COPY requirements.txt . && pip install && COPY . .` переиспользует слой зависимостей, пока реально не изменится `requirements.txt`.
- **`--no-cache-dir`** — pip's `~/.cache/pip` is tens of MB of wheels you never need at runtime. Layers are additive — deleting the cache in a later `RUN` does NOT remove it from the image; it just hides it behind another layer.
  * **`--no-cache-dir`** — каталог pip `~/.cache/pip` — это десятки мегабайт wheel-файлов, которые в рантайме не нужны. Слои только добавляются — удаление кеша в последующем `RUN` НЕ убирает его из образа, а лишь прячет за другим слоем.
- **Exec form `CMD`** — Shell form: `/bin/sh -c "python app.py"` → sh is PID 1, your app is PID 2, signals never reach you. Exec form `["python","app.py"]` → your app *is* PID 1.
  * **`CMD` в exec-форме** — shell-форма: `/bin/sh -c "python app.py"` → PID 1 становится sh, твоё приложение — PID 2, и сигналы до него не доходят. Exec-форма `["python","app.py"]` → твоё приложение *само* становится PID 1.
- **`.dockerignore`** — Toyota 2022: customer data leaked because `.git/` was packaged into a deployed image. Standard mitigation.
  * **`.dockerignore`** — Toyota в 2022 году: данные клиентов утекли, потому что `.git/` попал в задеплоенный образ. Стандартная мера защиты.

References: [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/) · [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)

Ссылки: [Лучшие практики Dockerfile](https://docs.docker.com/build/building/best-practices/) · [Справочник по Dockerfile](https://docs.docker.com/reference/dockerfile/)

</details>

---

## Task 2 — Build, Run, and Verify (2 pts)

Задание 2 — Собери, запусти и проверь (2 балла)

### 2.1 — Build, run, hit the endpoints

2.1 — Сборка, запуск и обращение к эндпоинтам

`YOUR TASK`: build the image, run it, and prove both endpoints respond exactly like Lab 1. Capture the real CLI output.

`YOUR TASK`: собери образ, запусти его и докажи, что оба эндпоинта отвечают так же, как в лабе 1. Сохрани реальный вывод CLI.

```bash
# YOUR TASK: build it
docker build -t devops-info-service:lab02 ./app_python

# YOUR TASK: run it (background, host:container port mapping is yours to choose)
docker run -d --rm -p ___:___ --name lab02 devops-info-service:lab02

# YOUR TASK: hit both endpoints and capture the JSON
curl -s http://localhost:___/ | jq .
curl -s http://localhost:___/health | jq -c .
```

Пояснение: `docker build` собирает образ с тегом `lab02`; `docker run -d` запускает контейнер в фоне с пробросом порта `хост:контейнер`; `curl ... | jq` проверяет оба эндпоинта и форматирует JSON. Подставь свои значения вместо `___`.

### 2.2 — Prove the container is non-root

2.2 — Докажи, что контейнер работает не от root

`YOUR TASK`: run the same image with `id -u` as the entrypoint and capture the output. **It must NOT print `0`.**

`YOUR TASK`: запусти тот же образ с `id -u` в качестве entrypoint и сохрани вывод. **Он НЕ должен выводить `0`.**

```bash
docker run --rm devops-info-service:lab02 id -u
# YOUR TASK: paste the captured number into docs/LAB02.md
```

Пояснение: команда переопределяет `CMD` образа и выполняет `id -u` — если вернётся `0`, контейнер работает от root, и это ошибка.

If you see `0`, your `USER ___` is missing or above the `COPY` that overwrites it. Re-order, rebuild, re-verify.

Если видишь `0`, значит `USER ___` отсутствует или стоит выше `COPY`, который его перезаписывает. Переставь директивы, пересобери и проверь заново.

### 2.3 — Inspect the artifact

2.3 — Изучи артефакт

```bash
docker images devops-info-service --format '{{.Tag}}\t{{.Size}}'
docker history devops-info-service:lab02
```

Пояснение: первая команда показывает тег и размер образа, вторая — историю слоёв. По ней видно, какой слой занимает больше всего места.

`YOUR TASK`: read your own `docker history` output. In `docs/LAB02.md`, identify the **single largest layer** and explain in one sentence why it's that big — base image, deps, or your source code?

`YOUR TASK`: разбери вывод собственной команды `docker history`. В `docs/LAB02.md` найди **самый большой слой** и объясни одним предложением, почему он такой большой — базовый образ, зависимости или твой исходный код?

### 2.4 — Proof of work

2.4 — Подтверждение работы

**Paste into `docs/LAB02.md`:**

**Вставь в `docs/LAB02.md`:**

- Build log tail (last ~10 lines including the `=> exporting to image` step)
  * Хвост лога сборки (примерно последние 10 строк, включая шаг `=> exporting to image`)
- Both endpoint JSONs from real `curl` runs
  * JSON-ответы обоих эндпоинтов из реальных запусков `curl`
- The `id -u` line proving non-root (illustrative — `10001`, your value will differ if you picked another UID)
  * Строка вывода `id -u`, подтверждающая не-root (пример — `10001`; твоё значение будет другим, если выберешь иной UID)
- `docker images` row showing your real size (illustrative — `~130 MB` is typical for a slim-based Flask app)
  * Строка `docker images` с реальным размером (пример — `~130 MB` типично для Flask-приложения на slim-образе)
- One-sentence "biggest layer" analysis
  * Анализ «самого большого слоя» в одном предложении

---

## Task 3 — Multi-Stage Build (2 pts)

Задание 3 — Многоступенчатая сборка (2 балла)

### 3.1 — Refactor to two stages

3.1 — Перепиши в две стадии

`YOUR TASK`: rewrite `app_python/Dockerfile` (or create `Dockerfile.multi`) as a multi-stage build. The **builder** stage installs deps into an isolated location; the **runtime** stage copies only what it needs and contains no `pip`, no build toolchain, no apt caches.

`YOUR TASK`: перепиши `app_python/Dockerfile` (или создай `Dockerfile.multi`) как многоступенчатую сборку. Стадия **builder** устанавливает зависимости в изолированное место; стадия **runtime** копирует только необходимое и не содержит ни `pip`, ни сборочного тулчейна, ни кешей apt.

```dockerfile
# syntax=docker/dockerfile:1
# ---------- Stage 1: builder ----------
FROM ___ AS builder           # YOUR TASK: same slim base (or a fatter one if you need gcc for wheels)
WORKDIR ___                   # YOUR TASK: pick a build location
COPY requirements.txt .
RUN ___                       # YOUR TASK: install deps into an ISOLATED dir — venv at /opt/venv OR pip --prefix=/install

# ---------- Stage 2: runtime ----------
FROM ___ AS runtime           # YOUR TASK: same slim base — minimal final surface
RUN ___                       # YOUR TASK: numeric non-root user again (stages don't share USER)
WORKDIR ___                   # YOUR TASK: absolute path
COPY --from=builder ___ ___   # YOUR TASK: copy ONLY the deps dir from the builder
ENV PATH="___:$PATH"          # YOUR TASK: prepend your venv/prefix bin dir
COPY ___ ___                  # YOUR TASK: app source LAST
USER ___                      # YOUR TASK: numeric UID
EXPOSE ___                    # YOUR TASK: same port as Task 1
CMD [___]                     # YOUR TASK: exec form
```

Пояснение: стадия `builder` ставит зависимости в отдельный каталог (`/opt/venv` или префикс `/install`), а стадия `runtime` копирует только этот каталог. Пользователь `USER` и рабочая директория задаются в каждой стадии заново — стадии не наследуют их друг от друга.

### 3.2 — Build and compare

3.2 — Собери и сравни

```bash
# If you replaced Dockerfile in-place:
docker build -t devops-info-service:lab02-multi ./app_python

# If you kept the single-stage as Dockerfile and put the multi-stage in Dockerfile.multi,
# pass it explicitly with -f (otherwise docker build picks the default Dockerfile):
docker build -f app_python/Dockerfile.multi -t devops-info-service:lab02-multi ./app_python

docker images devops-info-service --format '{{.Tag}}\t{{.Size}}'
```

Пояснение: если многоступенчатый Dockerfile лежит в отдельном файле, его нужно указать флагом `-f`. Финальная команда покажет теги и размеры обоих образов для сравнения.

`YOUR TASK`: capture both rows (`lab02` vs `lab02-multi`) and write a **one-paragraph** finding in `docs/LAB02.md`.

`YOUR TASK`: сохрани обе строки (`lab02` и `lab02-multi`) и напиши **один абзац** с выводом в `docs/LAB02.md`.

> ⚠️ **Honest finding you might hit:** for a **pure-Python** Flask app, the multi-stage image can be *slightly larger* (e.g. 133 vs 130 MB) than the single-stage — Flask has no compiled wheels, so there's no toolchain to discard, and the venv copy adds a few MB of metadata. **That's expected.** Document it; don't fake a size drop you didn't measure. The dramatic win comes for **compiled languages** (the Bonus) — there, multi-stage shrinks 60×.
>
> ⚠️ **Честный результат, с которым можно столкнуться:** для Flask-приложения на **чистом Python** многоступенчатый образ может оказаться *чуть больше* (например, 133 против 130 МБ), чем одноступенчатый — у Flask нет скомпилированных wheel, значит и отбрасывать нечего, а копирование venv добавляет несколько мегабайт метаданных. **Это ожидаемо.** Задокументируй это; не выдумывай уменьшение размера, которого не измерял. Драматический выигрыш появляется на **компилируемых языках** (бонус) — там многоступенчатая сборка уменьшает образ в 60 раз.

### 3.3 — Proof of work

3.3 — Подтверждение работы

**Paste into `docs/LAB02.md`:**

**Вставь в `docs/LAB02.md`:**

- Your `Dockerfile.multi` contents
  * Содержимое твоего `Dockerfile.multi`
- `docker images` rows showing your real `lab02` vs `lab02-multi` sizes
  * Строки `docker images` с реальными размерами `lab02` и `lab02-multi`
- One paragraph: did multi-stage shrink the image, leave it about the same, or grow it slightly? Why? (If your answer doesn't reference compiled wheels / build toolchain, you didn't understand the lecture.)
  * Один абзац: многоступенчатая сборка уменьшила образ, оставила примерно тем же или немного увеличила? Почему? (Если в ответе нет ссылки на скомпилированные wheel / сборочный тулчейн, значит лекция не усвоена.)
- `curl -s localhost:___/health` from the multi-stage container — same response as Task 2
  * `curl -s localhost:___/health` из контейнера, собранного многоступенчато, — такой же ответ, как в задании 2

<details>
<summary>💡 Multi-stage concepts</summary>

💡 Понятия многоступенчатой сборки

- **Why two stages?** Builder may need a C compiler to build a wheel (e.g. `cryptography`, `psycopg2`). Runtime never does. `COPY --from=builder` is how you discard everything the runtime doesn't need.
  * **Зачем две стадии?** Сборщику может понадобиться C-компилятор для сборки wheel (например, `cryptography`, `psycopg2`). В рантайме он не нужен никогда. `COPY --from=builder` позволяет отбросить всё, что не требуется в рантайме.
- **Naming stages.** `FROM image AS builder`, then `COPY --from=builder /src /dst`. The `AS` name is the handle.
  * **Именование стадий.** `FROM image AS builder`, затем `COPY --from=builder /src /dst`. Имя после `AS` — это идентификатор стадии.
- **Where this shines.** Compiled languages. `golang:1.25` ≈ 900 MB; multi-stage to `distroless/static` ≈ 15 MB. Same binary, 60× smaller, zero shell, zero package manager. That's the Bonus.
  * **Где это особенно заметно.** На компилируемых языках. `golang:1.25` ≈ 900 МБ; многоступенчатая сборка с `distroless/static` ≈ 15 МБ. Тот же бинарник, в 60 раз меньше, без shell и без пакетного менеджера. Это и есть бонус.

Reference: [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

Ссылка: [Многоступенчатые сборки](https://docs.docker.com/build/building/multi-stage/)

</details>

---

## Task 4 — Scan with Trivy (2 pts)

Задание 4 — Сканирование в Trivy (2 балла)

### 4.1 — Full scan + gated scan

4.1 — Полное сканирование и сканирование с проверкой

`YOUR TASK`: run two scans against your multi-stage image and capture both outputs.

`YOUR TASK`: запусти два сканирования многоступенчатого образа и сохрани оба вывода.

```bash
# Full informational scan — lists every CVE by severity
trivy image devops-info-service:lab02-multi

# CI-style gate — fails (non-zero exit) on HIGH/CRITICAL findings.
# This is the exact gate you'll add to GitHub Actions in Lab 3.
trivy image --severity HIGH,CRITICAL --exit-code 1 devops-info-service:lab02-multi
echo "exit code: $?"
```

Пояснение: первая команда выводит все найденные CVE с разбивкой по критичности; вторая завершится с ненулевым кодом при наличии HIGH/CRITICAL, поэтому подходит для CI-проверки. `echo` показывает код возврата последней команды.

> 🌐 If the DB pull dies with `ghcr.io/aquasecurity/trivy-db: DENIED`, add `--db-repository public.ecr.aws/aquasecurity/trivy-db:2`. AWS hosts an anonymous mirror — Aqua publishes there explicitly because ghcr.io anonymous tokens get rate-limited.
>
> 🌐 Если загрузка базы падает с `ghcr.io/aquasecurity/trivy-db: DENIED`, добавь `--db-repository public.ecr.aws/aquasecurity/trivy-db:2`. AWS размещает анонимное зеркало — Aqua публикует его специально, потому что анонимные токены ghcr.io упираются в rate limit.

### 4.2 — Severity table

4.2 — Таблица критичности

`YOUR TASK`: fill in your **real** counts in `docs/LAB02.md` (the numbers below are illustrative — yours will differ depending on base-image freshness):

`YOUR TASK`: внеси **реальные** числа в `docs/LAB02.md` (числа ниже — пример; твои будут отличаться в зависимости от свежести базового образа):

| Severity  | Count *(illustrative — your numbers will differ)* |
|-----------|--------:|
| CRITICAL  | 2 |
| HIGH      | 5 |
| MEDIUM    | 33 |
| LOW       | 63 |
| UNKNOWN   | 5 |
| **Total** | **108** |

| Критичность | Количество *(пример — твои числа будут отличаться)* |
|-----------|--------:|
| CRITICAL  | 2 |
| HIGH      | 5 |
| MEDIUM    | 33 |
| LOW       | 63 |
| UNKNOWN   | 5 |
| **Всего** | **108** |

### 4.3 — Explain one finding + remediate

4.3 — Объясни одну находку и устрани её

`YOUR TASK`: pick **one** finding (ideally a HIGH or CRITICAL — easier to defend the priority). In `docs/LAB02.md` write **at least 3 sentences** covering:

`YOUR TASK`: выбери **одну** находку (лучше HIGH или CRITICAL — проще обосновать приоритет). В `docs/LAB02.md` напиши **как минимум 3 предложения**, покрывающих:

1. The package + CVE id (e.g. `libncursesw6 6.5+20250216-2 — CVE-2025-69720`)
   * Пакет и идентификатор CVE (например, `libncursesw6 6.5+20250216-2 — CVE-2025-69720`)
2. Why it's flagged — what the upstream advisory says
   * Почему он отмечен — что говорит рекомендация производителя
3. What you'd do — rebuild on a fresh `python:3.13-slim` (often fixes itself when Debian patches land), pin a fixed library version, switch to a smaller base (distroless has near-zero CVEs — see the Bonus), or document it as **unfixable today** (no upstream patch yet)
   * Что ты сделаешь — пересоберёшь на свежем `python:3.13-slim` (часто проблема исчезает после патчей Debian), зафиксируешь исправленную версию библиотеки, перейдёшь на меньший базовый образ (у distroless почти нет CVE — см. бонус) или задокументируешь её как **неисправимую на сегодня** (патча от производителя ещё нет)

> 🚫 **Do NOT suppress findings via `.trivyignore`** to make the gate pass. If a CVE has no fix yet, document it explicitly. Hiding unfixed CVEs from CI is how you ship the next Log4Shell to prod.
>
> 🚫 **НЕ подавляй находки через `.trivyignore`**, чтобы пройти проверку. Если для CVE ещё нет исправления, задокументируй её явно. Скрывать неисправленные CVE от CI — так и выкатывают в прод следующий Log4Shell.

### 4.4 — Proof of work

4.4 — Подтверждение работы

**Paste into `docs/LAB02.md`:**

**Вставь в `docs/LAB02.md`:**

- Full Trivy output (or a screenshot — long, but the table at the top is the load-bearing part)
  * Полный вывод Trivy (или скриншот — он длинный, но главное — таблица вверху)
- Your severity table with real numbers
  * Твоя таблица критичности с реальными числами
- The gated scan command + its exit code
  * Команда сканирования с проверкой и её код возврата
- Your one-finding explanation and remediation reasoning
  * Объяснение одной находки и рассуждение об устранении

<details>
<summary>💡 Why scan, why gate on exit code</summary>

💡 Зачем сканировать и зачем проверять код возврата

- `trivy image <ref>` reports CVEs against OS packages **and** language dependencies (e.g. `requirements.txt`).
  * `trivy image <ref>` находит CVE в пакетах ОС **и** в зависимостях языка (например, в `requirements.txt`).
- `--severity HIGH,CRITICAL --exit-code 1` is the "fail the pipeline" idiom — by ignoring LOW/MEDIUM you keep the gate noise-free while still catching the dangerous stuff.
  * `--severity HIGH,CRITICAL --exit-code 1` — идиома «завалить пайплайн»: игнорируя LOW/MEDIUM, ты избавляешь проверку от шума, но всё равно ловишь опасное.
- Smaller base → fewer packages → fewer CVEs. Slim → distroless → scratch is a vulnerability-reduction ladder. The Bonus shows this concretely (Python slim ≈ 108 CVEs vs Go distroless ≈ 0).
  * Меньше базовый образ → меньше пакетов → меньше CVE. Slim → distroless → scratch — это лестница снижения уязвимости. Бонус показывает это наглядно (Python slim ≈ 108 CVE против Go distroless ≈ 0).

Reference: [Trivy docs](https://trivy.dev/) · [Trivy `image` command](https://trivy.dev/latest/docs/target/container_image/)

Ссылки: [Документация Trivy](https://trivy.dev/) · [команда `image` в Trivy](https://trivy.dev/latest/docs/target/container_image/)

</details>

---

## Task 5 — Push to a Registry & Document (1 pt)

Задание 5 — Запушь образ в реестр и задокументируй (1 балл)

### 5.1 — Tag + push

5.1 — Тег и пуш

`YOUR TASK`: publish your image to **GHCR** (recommended — free for public, integrates with the Actions you'll write in Lab 3) or Docker Hub.

`YOUR TASK`: опубликуй образ в **GHCR** (рекомендуется — бесплатно для публичных пакетов и хорошо сочетается с Actions, которые ты напишешь в лабе 3) или в Docker Hub.

For **GHCR** (recommended): create a Personal Access Token (classic) with the `write:packages` scope, then:

Для **GHCR** (рекомендуется): создай Personal Access Token (classic) с областью `write:packages`, затем:

```bash
# Put your token in a shell variable FIRST — do NOT paste the token onto the docker login
# line (it would land in your shell history). --password-stdin reads from stdin only.
export GHCR_PAT='ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

echo "$GHCR_PAT" | docker login ghcr.io -u <github-username> --password-stdin
docker tag devops-info-service:lab02-multi ghcr.io/<github-username>/devops-info-service:___   # YOUR TASK: real version tag
docker push ghcr.io/<github-username>/devops-info-service:___                                  # YOUR TASK: same tag
```

Пояснение: токен сначала кладётся в переменную окружения и передаётся в `docker login` через `--password-stdin`, чтобы не оставить его в истории shell. Затем образ получает тег с версией и публикуется через `docker push`; тег должен совпадать в обеих командах.

> 🤖 **In CI (Lab 3), don't use a PAT.** GitHub Actions provides an ephemeral `GITHUB_TOKEN` per run that has `write:packages` for the repo's own GHCR namespace. The Lab 3 docker job uses
> `password: ${{ secrets.GITHUB_TOKEN }}` — no long-lived PAT to leak. PATs are only for *your laptop today*.
>
> 🤖 **В CI (лаба 3) не используй PAT.** GitHub Actions выдаёт на каждый запуск временный `GITHUB_TOKEN` с правами `write:packages` для GHCR-пространства самого репозитория. Docker-джоба в лабе 3 использует
> `password: ${{ secrets.GITHUB_TOKEN }}` — долгоживущему PAT негде утечь. PAT нужен только для *твоего ноутбука сегодня*.

> 📢 **GHCR packages are PRIVATE by default after first push.** The Acceptance Criteria below require *publicly pullable* — after `docker push` succeeds the first time, go to **your GitHub profile → Packages → `devops-info-service` → Package settings (right side) → Change visibility → Public**. Confirm by typing the package name. Without this step the `docker pull` from a clean cache below will work only because your login is still cached; a grader running `docker pull` without your token will get `401 Unauthorized`.
>
> 📢 **Пакеты в GHCR после первого пуша по умолчанию приватные.** Критерии приёмки ниже требуют, чтобы образ можно было *публично скачать*: после первого успешного `docker push` перейди в **профиль GitHub → Packages → `devops-info-service` → Package settings (справа) → Change visibility → Public**. Подтверди, введя имя пакета. Без этого шага `docker pull` из чистого кеша ниже сработает только потому, что твой логин ещё закеширован; проверяющий, запустивший `docker pull` без твоего токена, получит `401 Unauthorized`.

`YOUR TASK`: pick a tagging strategy and put the real tag in both blanks above. **Not `:latest`** — see slide 20. Pick one:

`YOUR TASK`: выбери стратегию тегирования и подставь реальный тег в оба пропуска выше. **Не `:latest`** — см. слайд 20. Выбери один:

| Strategy   | Example       | When to use |
|------------|---------------|-------------|
| SemVer     | `1.0.0`       | A real release of the service |
| Git SHA    | `git-3a7f29c` | CI builds; always unique |
| CalVer     | `2026.05.0`   | Time-driven release cadence |

| Стратегия  | Пример        | Когда использовать |
|------------|---------------|--------------------|
| SemVer     | `1.0.0`       | Реальный релиз сервиса |
| Git SHA    | `git-3a7f29c` | CI-сборки; всегда уникальны |
| CalVer     | `2026.05.0`   | Релизы по календарю |

Verify it's actually pullable from a **truly clean state** — `docker logout` first so cached credentials don't mask a private-package mistake:

Проверь, что образ действительно скачивается из **полностью чистого состояния** — сначала `docker logout`, чтобы закешированные учётные данные не скрыли ошибку с приватным пакетом:

```bash
docker logout ghcr.io                                            # forget the PAT so we test the public path
docker rmi  ghcr.io/<github-username>/devops-info-service:___   # YOUR TASK: your tag
docker pull ghcr.io/<github-username>/devops-info-service:___   # YOUR TASK: must succeed anonymously (package must be Public)
```

Пояснение: `docker logout` и `docker rmi` удаляют локальные учётные данные и копию образа, после чего `docker pull` должен отработать анонимно. Успешная загрузка доказывает, что пакет публичный.

If `docker pull` returns `unauthorized` after `docker logout`, your package is still private — flip its visibility to Public per the note above, then re-pull.

Если после `docker logout` команда `docker pull` возвращает `unauthorized`, пакет всё ещё приватный — поменяй видимость на Public по заметке выше и повтори загрузку.

### 5.2 — Update `app_python/README.md`

5.2 — Обнови `app_python/README.md`

`YOUR TASK`: add a **Docker** section to your Lab 1 README with three subsections:

`YOUR TASK`: добавь в README из лабы 1 раздел **Docker** с тремя подразделами:

1. **Build** — your `docker build` command
   * **Build** — твоя команда `docker build`
2. **Run** — your `docker run` command (host:container port mapping)
   * **Run** — твоя команда `docker run` (проброс порта хост:контейнер)
3. **Pull from registry** — the public pull command using your real registry URL
   * **Pull from registry** — команда публичного скачивания с твоим реальным URL реестра

No need to paste the Dockerfile contents into the README — link to the file in the repo instead.

Не нужно вставлять содержимое Dockerfile в README — лучше дай ссылку на файл в репозитории.

### 5.3 — `app_python/docs/LAB02.md` — submission report

5.3 — `app_python/docs/LAB02.md` — отчёт о сдаче

Required sections (in order):

Обязательные разделы (по порядку):

1. **Dockerfile decisions** — your Task 1 file + a *why* line per `YOUR TASK`
   * **Решения по Dockerfile** — файл из задания 1 и строка *почему* для каждого `YOUR TASK`
2. **Multi-stage results** — single-stage vs multi-stage size with real numbers and a one-line takeaway (including the honest "no change because pure Python" answer if that's what you measured)
   * **Результаты многоступенчатой сборки** — размеры одно- и многоступенчатого образа с реальными числами и вывод в одну строку (включая честный ответ «без изменений, потому что чистый Python», если ты получил именно это)
3. **Trivy scan** — severity table with real counts, one finding explained, remediation reasoning, gated-scan exit code
   * **Сканирование Trivy** — таблица критичности с реальными числами, разбор одной находки, рассуждение об устранении, код возврата сканирования с проверкой
4. **Build / run / push evidence** — the captured outputs from Tasks 2 and 5, including the public registry URL
   * **Подтверждения сборки / запуска / пуша** — сохранённые выводы из заданий 2 и 5, включая публичный URL реестра
5. **Challenges & Solutions** — at least one real one (the `slim has no curl` healthcheck trap, or the Trivy DB mirror, are both fair game — pick what you actually hit)
   * **Трудности и решения** — как минимум одна реальная (ловушка `slim has no curl` в healthcheck или зеркало базы Trivy — обе подойдут; выбирай то, с чем действительно столкнулся)

### 5.4 — Proof of work

5.4 — Подтверждение работы

**Paste into `docs/LAB02.md`:**

**Вставь в `docs/LAB02.md`:**

- `docker push` output showing the digest
  * Вывод `docker push` с digest'ом
- `docker pull` output from the clean state (proves the image is public and pullable)
  * Вывод `docker pull` из чистого состояния (доказывает, что образ публичный и скачивается)
- The registry URL (e.g. `https://github.com/<you>/devops-info-service/pkgs/container/devops-info-service`)
  * URL реестра (например, `https://github.com/<you>/devops-info-service/pkgs/container/devops-info-service`)
- The Docker section of your updated `README.md` (or a link to it in the PR)
  * Раздел Docker из обновлённого `README.md` (или ссылка на него в PR)

---

## Bonus Task — Distroless Multi-Stage (Go) (2 pts)

Бонусное задание — многоступенчатая сборка Distroless (Go) (2 балла)

Re-containerize your Lab 1 bonus app (Go recommended) as a multi-stage build with a **distroless or `scratch`** final stage. This is where the lecture's "60× smaller, zero CVEs" payoff actually shows up.

Заново упакуй в контейнер бонусное приложение из лабы 1 (рекомендуется Go) как многоступенчатую сборку с финальной стадией **distroless или `scratch`**. Именно здесь проявляется обещанный на лекции результат «в 60 раз меньше, ноль CVE».

`YOUR TASK`: fill the skeleton. Less hand-holding than the main tasks — by now you've written two Dockerfiles.

`YOUR TASK`: заполни скелет. Здесь меньше подсказок, чем в основных заданиях, — к этому моменту ты уже написал два Dockerfile.

```dockerfile
# syntax=docker/dockerfile:1
# ---------- build stage: full Go toolchain ----------
FROM ___ AS build             # YOUR TASK: pin a Go base (e.g. golang:1.25). Not :latest
WORKDIR ___                   # YOUR TASK: pick a build dir
COPY go.mod go.sum* ./
RUN go mod download
COPY . .
RUN ___                       # YOUR TASK: build a FULLY STATIC binary. Which env var disables cgo? (slide 17)
                              # WHY this matters: cgo links against the build image's libc dynamically.
                              # A static binary (no libc dep) is what makes `FROM scratch` / distroless/static
                              # viable — those images literally don't ship libc.

# ---------- final stage: ~nothing but the binary ----------
FROM ___                      # YOUR TASK: gcr.io/distroless/static-debian12 OR scratch — choose + defend in docs
COPY --from=build ___ ___     # YOUR TASK: copy the static binary out of the build stage
USER ___                      # YOUR TASK: distroless ships `nonroot:nonroot`; or pick a numeric UID
EXPOSE ___                    # YOUR TASK: same port as Lab 1
ENTRYPOINT [___]              # YOUR TASK: exec form — single static binary, no shell needed
```

Пояснение: стадия сборки использует полный тулчейн Go и собирает полностью статический бинарник (`CGO_ENABLED=0`), а финальная стадия содержит только этот бинарник. Поэтому `scratch` или `distroless/static` работают: в них нет libc, но статическому бинарнику она и не нужна.

### Bonus requirements

Требования к бонусу

- Multi-stage Dockerfile in `app_go/` with a distroless or `scratch` final stage
  * Многоступенчатый Dockerfile в `app_go/` с финальной стадией distroless или `scratch`
- Statically-linked binary — explain in `app_go/docs/LAB02.md` *why* static linking is what makes distroless/scratch viable (no libc at runtime)
  * Статически слинкованный бинарник — объясни в `app_go/docs/LAB02.md`, *почему* именно статическая линковка делает distroless/scratch жизнеспособными (в рантайме нет libc)
- Working containerized app — both `/` and `/health` respond
  * Работающее приложение в контейнере — отвечают и `/`, и `/health`
- Trivy scan of the distroless image — compare its finding count to the Python multi-stage image and explain *why* it's so much lower (no shell, no package manager, near-zero attack surface)
  * Сканирование distroless-образа в Trivy — сравни число находок с многоступенчатым Python-образом и объясни, *почему* их гораздо меньше (нет shell, нет пакетного менеджера, почти нулевая поверхность атаки)
- Size + CVE comparison documented (the lecture predicts ~15 MB / ~0 CVEs — capture your real numbers)
  * Задокументированное сравнение размера и CVE (лекция предсказывает ~15 МБ / ~0 CVE — зафиксируй свои реальные числа)

**Expected payoff** *(illustrative — your numbers will differ)*: Python slim ≈ **130 MB / ~100 CVEs** → Go distroless ≈ **8 MB / 0 CVEs**.

**Ожидаемый результат** *(пример — твои числа будут отличаться)*: Python slim ≈ **130 МБ / ~100 CVE** → Go distroless ≈ **8 МБ / 0 CVE**.

<details>
<summary>💡 Distroless vs scratch — when each fits</summary>

💡 Distroless против scratch — когда что подходит

- **`gcr.io/distroless/static-debian12`** — empty userspace except CA certs + `/etc/passwd` (a `nonroot` user) + `tzdata`. Recommended default for Go.
  * **`gcr.io/distroless/static-debian12`** — пустое пользовательское пространство, кроме CA-сертификатов, `/etc/passwd` (пользователь `nonroot`) и `tzdata`. Рекомендуемый вариант по умолчанию для Go.
- **`scratch`** — empty. ~0 CVEs because there's nothing in the image. Works only for fully static binaries. You'd add CA certs manually if your binary calls HTTPS.
  * **`scratch`** — полностью пустой. ~0 CVE, потому что в образе ничего нет. Подходит только для полностью статических бинарников. Если бинарник ходит по HTTPS, CA-сертификаты придётся добавить вручную.
- **Trade-off** — no shell means no `docker exec ... sh` for debugging in production. That's a *feature* under attack and an *annoyance* during dev. Keep a `*-debug` distroless variant around if you need it locally.
  * **Компромисс** — без shell не зайти через `docker exec ... sh` для отладки в проде. При атаке это *фича*, при разработке — *неудобство*. Держи под рукой debug-вариант distroless (`*-debug`), если он нужен локально.

Reference: [distroless](https://github.com/GoogleContainerTools/distroless) · [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

Ссылки: [distroless](https://github.com/GoogleContainerTools/distroless) · [Многоступенчатые сборки](https://docs.docker.com/build/building/multi-stage/)

</details>

---

## How to Submit

Как сдавать

```bash
git switch -c lab02
git add app_python/Dockerfile app_python/.dockerignore app_python/README.md app_python/docs/LAB02.md
git add app_go/                   # only if you did the bonus
git commit -m "feat(lab02): containerize devops info service (multi-stage + trivy + push)"
git push -u origin lab02
```

Пояснение: создай ветку `lab02`, добавь в индекс Dockerfile, `.dockerignore`, обновлённый README и отчёт, а затем закоммить и запушь ветку. Строку с `app_go/` добавляй только при выполнении бонуса.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab02` → `course-repo:master` *(reviewed)*
  * `your-fork:lab02` → `course-repo:master` *(с ревью)*
- `your-fork:lab02` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab02` → `your-fork:master` *(сливается в твою основную ветку по завершении)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 — Dockerfile + .dockerignore written by hand
- [ ] Task 2 — build / run / non-root verified with real curl + id -u captures
- [ ] Task 3 — multi-stage Dockerfile with size comparison
- [ ] Task 4 — Trivy full + gated scan; one finding explained
- [ ] Task 5 — image pushed to GHCR/Docker Hub with SemVer or SHA tag (not :latest only)
- [ ] Bonus  — Go distroless image, ~order-of-magnitude smaller, ~0 CVEs (if attempted)
```

Пояснение: отмечай пункты по мере выполнения — это краткая сводка всех заданий и бонуса.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Dockerfile (3 pts)

Задание 1 — Dockerfile (3 балла)

- ✅ `app_python/Dockerfile` written from the skeleton (every `___` resolved)
  * ✅ `app_python/Dockerfile` написан по скелету (каждый `___` заполнен)
- ✅ Pinned slim base (not `latest`, not the full image)
  * ✅ Зафиксированный slim-базовый образ (не `latest` и не полный образ)
- ✅ `WORKDIR` set explicitly
  * ✅ `WORKDIR` задан явно
- ✅ **Numeric** non-root `USER` (uid ≥ 10000)
  * ✅ **Числовой** непривилегированный `USER` (uid ≥ 10000)
- ✅ Dependencies installed before app code (cache-friendly order); `--no-cache-dir`
  * ✅ Зависимости устанавливаются до кода приложения (порядок, дружелюбный к кешу); `--no-cache-dir`
- ✅ `CMD` in exec form (JSON array)
  * ✅ `CMD` в exec-форме (JSON-массив)
- ✅ `app_python/.dockerignore` present with VCS / venv / cache / IDE exclusions
  * ✅ `app_python/.dockerignore` присутствует и исключает VCS / venv / кеши / файлы IDE

### Task 2 — Build & Run (2 pts)

Задание 2 — Сборка и запуск (2 балла)

- ✅ Image builds clean
  * ✅ Образ собирается без ошибок
- ✅ `GET /` and `GET /health` return the same JSON as Lab 1
  * ✅ `GET /` и `GET /health` возвращают тот же JSON, что в лабе 1
- ✅ `docker run ... id -u` prints a non-zero UID
  * ✅ `docker run ... id -u` выводит ненулевой UID
- ✅ `docker images` size + "biggest layer" one-liner in docs
  * ✅ Размер из `docker images` и однострочный вывод про «самый большой слой» в документации

### Task 3 — Multi-stage (2 pts)

Задание 3 — Многоступенчатая сборка (2 балла)

- ✅ Two stages with `COPY --from=builder`
  * ✅ Две стадии с `COPY --from=builder`
- ✅ No build toolchain / pip cache in the final image
  * ✅ В финальном образе нет сборочного тулчейна и кеша pip
- ✅ Real size numbers in docs — including the **honest** explanation if multi-stage didn't shrink it for your pure-Python app
  * ✅ Реальные числа размеров в документации — включая **честное** объяснение, если многоступенчатая сборка не уменьшила образ твоего приложения на чистом Python

### Task 4 — Trivy (2 pts)

Задание 4 — Trivy (2 балла)

- ✅ Full scan + gated `--severity HIGH,CRITICAL --exit-code 1` both run
  * ✅ Выполнены и полное сканирование, и проверка `--severity HIGH,CRITICAL --exit-code 1`
- ✅ Severity counts with real numbers
  * ✅ Количество по критичности с реальными числами
- ✅ One finding explained with remediation reasoning
  * ✅ Одна находка разобрана с рассуждением об устранении
- ✅ No findings hidden via `.trivyignore`
  * ✅ Ни одна находка не скрыта через `.trivyignore`

### Task 5 — Registry & Docs (1 pt)

Задание 5 — Реестр и документация (1 балл)

- ✅ Image pushed and **publicly pullable** (`docker rmi && docker pull` from clean works)
  * ✅ Образ запушен и **публично скачивается** (`docker rmi && docker pull` из чистого состояния работает)
- ✅ Real version tag (SemVer or SHA), not only `:latest`
  * ✅ Реальный тег версии (SemVer или SHA), а не только `:latest`
- ✅ `README.md` Docker section
  * ✅ Раздел Docker в `README.md`
- ✅ `docs/LAB02.md` complete (all 5 sections, real evidence, registry URL)
  * ✅ `docs/LAB02.md` заполнен полностью (все 5 разделов, реальные подтверждения, URL реестра)

### Bonus (2 pts)

Бонус (2 балла)

- ✅ Multi-stage Go Dockerfile, distroless or `scratch` final stage
  * ✅ Многоступенчатый Dockerfile для Go с финальной стадией distroless или `scratch`
- ✅ Statically-linked binary; both endpoints work
  * ✅ Статически слинкованный бинарник; работают оба эндпоинта
- ✅ Trivy scan of distroless image with finding-count comparison
  * ✅ Сканирование distroless-образа в Trivy со сравнением числа находок
- ✅ `app_go/docs/LAB02.md` with size metrics, static-linking explanation, security analysis
  * ✅ `app_go/docs/LAB02.md` с метриками размера, объяснением статической линковки и анализом безопасности

---

## Rubric

Рубрика

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Dockerfile  | **3** | Pinned slim base, numeric non-root, cache-friendly layers, exec-form `CMD`, `.dockerignore` |
| **Task 2** — Build & Run | **2** | Builds clean; both endpoints match Lab 1; non-root verified |
| **Task 3** — Multi-stage | **2** | Two stages, no toolchain in final image, documented size reality (even if it's "no change, here's why") |
| **Task 4** — Trivy       | **2** | Full + gated scan, real counts, one explained finding, nothing hidden |
| **Task 5** — Registry    | **1** | Publicly pullable, real version tag, complete `LAB02.md` + README Docker section |
| **Bonus** — Distroless Go | **2** | Distroless/scratch multi-stage with size + CVE comparison |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Dockerfile  | **3** | Зафиксированный slim-базовый образ, числовой не-root пользователь, слои, дружелюбные к кешу, `CMD` в exec-форме, `.dockerignore` |
| **Задание 2** — Сборка и запуск | **2** | Собирается без ошибок; оба эндпоинта совпадают с лабой 1; не-root подтверждён |
| **Задание 3** — Многоступенчатая сборка | **2** | Две стадии, в финальном образе нет тулчейна, задокументирована реальность размеров (даже если это «без изменений, вот почему») |
| **Задание 4** — Trivy       | **2** | Полное и проверочное сканирование, реальные числа, одна разобранная находка, ничего не скрыто |
| **Задание 5** — Реестр      | **1** | Публично скачиваемый образ, реальный тег версии, полный `LAB02.md` и раздел Docker в README |
| **Бонус** — Distroless Go   | **2** | Многоступенчатая сборка Distroless/scratch со сравнением размера и CVE |
| **Всего** | **12** | 10 основных + 2 бонусных |

**Grading guidance:**

**Как оценивается:**

- **10/10** — Small, secure, reproducible image; multi-stage discipline; scan understood; clean push; reasoning in docs goes beyond "the lecture said to."
  * **10/10** — Маленький, безопасный, воспроизводимый образ; дисциплина многоступенчатой сборки; понимание сканирования; аккуратный пуш; рассуждения в документации выходят за рамки «так сказали на лекции».
- **8–9/10** — Working multi-stage image as non-root, scanned and pushed, defends most choices.
  * **8–9/10** — Рабочий многоступенчатый образ под не-root, просканирован и запушен, большинство решений обосновано.
- **6–7/10** — Container works and runs as non-root, but single-stage only or shallow scan/analysis.
  * **6–7/10** — Контейнер работает и запускается от не-root, но сборка только одноступенчатая или анализ/сканирование поверхностны.
- **<6/10** — Runs as root, no `.dockerignore`, no scan, or copy-paste Dockerfile the student can't explain.
  * **<6/10** — Работа от root, нет `.dockerignore`, нет сканирования или Dockerfile скопирован и не может быть объяснён.

---

## Resources

Материалы

<details>
<summary>📚 Docker Documentation</summary>

📚 Документация Docker

- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/)
  * [Лучшие практики Dockerfile](https://docs.docker.com/build/building/best-practices/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
  * [Справочник по Dockerfile](https://docs.docker.com/reference/dockerfile/)
- [Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
  * [Многоступенчатые сборки](https://docs.docker.com/build/building/multi-stage/)
- [`.dockerignore`](https://docs.docker.com/reference/dockerfile/#dockerignore-file)
  * [Файл `.dockerignore`](https://docs.docker.com/reference/dockerfile/#dockerignore-file)
- [Docker Engine 29 release notes](https://docs.docker.com/engine/release-notes/)
  * [Примечания к релизу Docker Engine 29](https://docs.docker.com/engine/release-notes/)
- [containerd image store](https://docs.docker.com/engine/storage/containerd/)
  * [Хранилище образов containerd](https://docs.docker.com/engine/storage/containerd/)

</details>

<details>
<summary>🔒 Security & Scanning</summary>

🔒 Безопасность и сканирование

- [Trivy docs](https://trivy.dev/) · [Trivy `image` scanning](https://trivy.dev/latest/docs/target/container_image/)
  * [Документация Trivy](https://trivy.dev/) · [Сканирование образов в Trivy](https://trivy.dev/latest/docs/target/container_image/)
- [Distroless](https://github.com/GoogleContainerTools/distroless)
  * [Distroless](https://github.com/GoogleContainerTools/distroless)
- [Why non-root containers](https://docs.docker.com/build/building/best-practices/#user)
  * [Зачем нужны контейнеры без root](https://docs.docker.com/build/building/best-practices/#user)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
  * [Бенчмарк CIS для Docker](https://www.cisecurity.org/benchmark/docker)
- *Container Security* — Liz Rice (O'Reilly, 2020)
  * *Container Security* — Лиз Райс (O'Reilly, 2020)

</details>

<details>
<summary>🛠️ Dev tools worth knowing</summary>

🛠️ Инструменты, которые стоит знать

- [Hadolint](https://github.com/hadolint/hadolint) — Dockerfile linter
  * [Hadolint](https://github.com/hadolint/hadolint) — линтер Dockerfile
- [Dive](https://github.com/wagoodman/dive) — explore image layers interactively
  * [Dive](https://github.com/wagoodman/dive) — интерактивный просмотр слоёв образа
- [GHCR](https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-container-registry) · [Docker Hub](https://hub.docker.com/)
  * [GHCR](https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-container-registry) · [Docker Hub](https://hub.docker.com/)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **`COPY . .` before `pip install`** — kills the layer cache. A one-character code change re-installs every dependency. The single most common mistake in this lab — fix it by copying `requirements.txt` first and `pip install`-ing before the rest of the source. (Lecture slide 14.)
  * **`COPY . .` до `pip install`** — убивает кеш слоёв. Изменение кода на один символ переустанавливает все зависимости. Самая частая ошибка в этой лабе — исправляется тем, что сначала копируешь `requirements.txt`, делаешь `pip install`, а уже потом копируешь остальные исходники. (Слайд 14 лекции.)
- **`USER appuser` (non-numeric) under `runAsNonRoot: true`** — works in Docker, fails the Lab 9 K8s admission policy because it can't verify the UID without parsing `/etc/passwd`. Always use a **numeric** `USER 10001`.
  * **`USER appuser` (не число) при `runAsNonRoot: true`** — работает в Docker, но не проходит политику допуска K8s в лабе 9: без разбора `/etc/passwd` она не может проверить UID. Всегда используй **числовой** `USER 10001`.
- **Trivy DB `ghcr.io ... DENIED`** — campus / restricted networks rate-limit the anonymous ghcr.io token. Use `--db-repository public.ecr.aws/aquasecurity/trivy-db:2` — Aqua publishes there explicitly for this case.
  * **Trivy DB `ghcr.io ... DENIED`** — университетские и ограниченные сети упираются в rate limit анонимного токена ghcr.io. Используй `--db-repository public.ecr.aws/aquasecurity/trivy-db:2` — Aqua публикует зеркало там специально для этого случая.
- **`python:3.13-slim` has no `curl` and no `wget`** — your HEALTHCHECK breaks silently. Either use `python -c 'import urllib.request; urllib.request.urlopen("http://127.0.0.1:8080/health")'` or `apt-get install curl` (and accept the extra CVEs). Don't add the line and assume it works.
  * **В `python:3.13-slim` нет `curl` и `wget`** — HEALTHCHECK молча ломается. Используй либо `python -c 'import urllib.request; urllib.request.urlopen("http://127.0.0.1:8080/health")'`, либо `apt-get install curl` (и прими дополнительные CVE). Не добавляй строку, предполагая, что она работает.
- **Trivy `v0.69.4` is malicious** — CVE-2026-33634 supply-chain compromise (March 2026). Pin `v0.69.3` or a later clean patch. In CI: `trivy-action@v0.35.0+`.
  * **Trivy `v0.69.4` вредоносна** — компрометация цепочки поставок CVE-2026-33634 (март 2026). Закрепи `v0.69.3` или более поздний чистый патч. В CI: `trivy-action@v0.35.0+`.
- **BuildKit not enabled** — older Docker Engines (<23) ignore `# syntax=docker/dockerfile:1` and skip `COPY --from=builder` features. Docker 29 has BuildKit on by default; if your build fails on `RUN --mount=type=cache` syntax, you're on an ancient Docker.
  * **BuildKit не включён** — старые Docker Engine (<23) игнорируют `# syntax=docker/dockerfile:1` и не поддерживают возможности вроде `COPY --from=builder`. В Docker 29 BuildKit включён по умолчанию; если сборка падает на синтаксисе `RUN --mount=type=cache`, у тебя очень древний Docker.
- **`.dockerignore` missing** — your `COPY . .` ships `.git/`, `.venv/`, IDE configs, and **`.env` with secrets** into the image. Layers are forever — once the secret is in the image, even deleting it in a later layer doesn't remove it.
  * **Нет `.dockerignore`** — твой `COPY . .` отправляет в образ `.git/`, `.venv/`, конфиги IDE и **`.env` с секретами**. Слои живут вечно: если секрет попал в образ, удаление в следующем слое его не убирает.
- **`COPY secrets.env / && rm secrets.env`** — does NOT remove the secret. It's still in the earlier layer, visible via `docker history` and `docker save | tar -xvf`. Treat every layer as immutable.
  * **`COPY secrets.env / && rm secrets.env`** — НЕ удаляет секрет. Он остаётся в предыдущем слое и виден через `docker history` и `docker save | tar -xvf`. Считай каждый слой неизменяемым.
- **Multi-stage marginally LARGER for pure Python** — Flask has no compiled wheels, so the builder stage has nothing to discard, and the venv copy adds ~3 MB. This is **expected and correct** — document it, don't hide it. The big multi-stage win is compiled languages (Bonus task).
  * **Многоступенчатая сборка чуть БОЛЬШЕ для чистого Python** — у Flask нет скомпилированных wheel, так что в стадии builder нечего отбрасывать, а копирование venv добавляет ~3 МБ. Это **ожидаемо и правильно** — задокументируй, не скрывай. Большой выигрыш от многоступенчатости — на компилируемых языках (бонус).
- **Image cache stale after `apt-get update`** — if you `RUN apt-get update && apt-get install foo` in two separate layers, the `update` is cached forever while `install` re-runs against a stale package list. Always combine them in one `RUN`. Same lesson, different package manager: don't split related work across layers.
  * **Кеш образа устаревает после `apt-get update`** — если разнести `RUN apt-get update` и `apt-get install foo` по двум слоям, `update` закешируется навсегда, а `install` будет работать со старым списком пакетов. Всегда объединяй их в одном `RUN`. Тот же урок для любого пакетного менеджера: не разделяй связанную работу по слоям.
- **Container hostname is the container ID, not your laptop** — `socket.gethostname()` inside the container returns something like `a3f29c8b1d04`. That's correct behaviour (Lab 1 footnote), not a bug. You're seeing the PID/UTS namespace at work.
  * **Имя хоста в контейнере — это ID контейнера, а не твой ноутбук** — `socket.gethostname()` внутри контейнера вернёт что-то вроде `a3f29c8b1d04`. Это корректное поведение (сноска в лабе 1), а не баг. Так работает пространство имён PID/UTS.
- **`docker run` as root by default** — even if your Dockerfile has `USER 10001`, `docker run --user root` overrides it. The Dockerfile sets the *default*; runtime can override. In Lab 9, Kubernetes PSA / Pod Security policies enforce this from the orchestrator side.
  * **`docker run` по умолчанию может запустить от root** — даже если в Dockerfile указан `USER 10001`, флаг `docker run --user root` его переопределяет. Dockerfile задаёт *значение по умолчанию*; рантайм может его переопределить. В лабе 9 это контролируют политики PSA / Pod Security со стороны оркестратора.

</details>

---

## Looking Ahead

Что дальше

The image you build this week follows your service through the rest of the course:

Образ, который ты соберёшь на этой неделе, будет сопровождать твой сервис до конца курса:

| Lab | What it adds |
|---:|---|
| 3  | CI: pytest + ruff + image build + Trivy gate on every PR |
| 7  | Structured JSON logs via Alloy → Loki |
| 8  | `/metrics` endpoint + Prometheus instrumentation |
| 9  | Deploy to k3d Kubernetes using the `/health` probe |
| 10 | Helm 4 chart packaging this image |
| 13 | ArgoCD GitOps shipping new tags automatically |

| Лаба | Что добавляет |
|---:|---|
| 3  | CI: pytest + ruff + сборка образа + Trivy-проверка на каждом PR |
| 7  | Структурированные JSON-логи через Alloy → Loki |
| 8  | Эндпоинт `/metrics` + инструментация Prometheus |
| 9  | Деплой в Kubernetes (k3d) с проверкой `/health` |
| 10 | Helm 4-чарт для упаковки этого образа |
| 13 | ArgoCD GitOps, автоматически выкатывающий новые теги |

---

**Hot take:** a good Dockerfile is small, secure, and reproducible. Two out of three is a bug. Run as non-root, scan every image, never deploy `:latest`. Everything else in this course is a variation on those three rules.

**Резко, но честно:** хороший Dockerfile — маленький, безопасный и воспроизводимый. Два из трёх — это баг. Запускай не от root, сканируй каждый образ, никогда не деплой `:latest`. Всё остальное в этом курсе — вариации этих трёх правил.
