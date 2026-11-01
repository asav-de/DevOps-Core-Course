# Lab 7 — Observability & Logging with the Loki Stack

Лаба 7 — наблюдаемость и логирование со стеком Loki

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Logging%20%26%20Observability-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Loki%20|%20Alloy%20|%20Grafana-informational)

> **Goal:** stand up Loki + Grafana Alloy + Grafana with Docker Compose, ship your Lab 1 Python app's logs into it as structured JSON, and learn to ask questions of those logs with LogQL.
> **Deliverable:** a PR from `lab07` adding `monitoring/` (compose stack + Alloy + Loki configs + dashboard JSON), the JSON-logging patch to `app_python/`, and `monitoring/docs/LAB07.md` with the evidence captures.
>
> **Цель:** поднять Loki + Grafana Alloy + Grafana через Docker Compose, отправлять логи твоего Python-приложения из лабы 1 в виде структурированного JSON и научиться задавать этим логам вопросы на LogQL.
> **Что сдать:** PR из ветки `lab07`, добавляющий `monitoring/` (compose-стек + конфиги Alloy и Loki + JSON дашборда), патч JSON-логирования для `app_python/` и `monitoring/docs/LAB07.md` с подтверждениями.

---

## Overview

Обзор

In this lab you will practice:

В этой лабе ты потренируешься:

- Reading Alloy's River syntax and **wiring components by name** (`discovery.docker` → `discovery.relabel` → `loki.source.docker` → `loki.write`) instead of copy-pasting a YAML pipeline
  * Разбираться в синтаксисе River в Alloy и **связывать компоненты по имени** (`discovery.docker` → `discovery.relabel` → `loki.source.docker` → `loki.write`), а не копировать готовый YAML-пайплайн
- Writing a Docker Compose stack from a service-list spec (which images, which ports, which volumes, which network)
  * Писать стек Docker Compose по спецификации списка сервисов (какие образы, порты, тома и сеть)
- Emitting structured JSON logs from a Python service
  * Выводить структурированные JSON-логи из Python-сервиса
- Writing **LogQL** stream selectors + line filters + parsers + metric functions to answer real questions about your own app
  * Писать селекторы потоков **LogQL**, фильтры строк, парсеры и метрические функции, чтобы отвечать на реальные вопросы о своём приложении
- Hardening the stack: resource limits, retention compactor, health checks, no anonymous Grafana
  * Укреплять стек: лимиты ресурсов, compactor для ретеншна, проверки здоровья, Grafana без анонимного доступа

> ⚠️ **Scope:** single-binary Loki on the local Docker host. No object storage, no multi-tenancy, no TLS — those come back in the K8s-era labs (12, 16). Don't tune for production; tune for *understanding the wires*.
>
> **Область:** однобинарный Loki на локальном Docker-хосте. Без объектного хранилища, мультитенантности и TLS — они вернутся в лабах эпохи Kubernetes (12 и 16). Не настраивай под прод — настраивай под *понимание связей*.

> 🪦 **A note on Promtail.** Through 2025 the standard Loki agent was **Promtail**. It reached **End-of-Life on March 2, 2026** — no further releases, no security fixes. Its successor, **Grafana Alloy**, folds Promtail + the Prometheus agent + the OpenTelemetry Collector into one binary. This lab uses Alloy. You may still meet Promtail configs in brownfield systems, so the lecture slides 14–15 show the legacy syntax and how it maps to Alloy — but **do not stand up a new Promtail in 2026.** A Promtail `pipeline_stages` block does **not** translate 1:1 to Alloy; the equivalents are split across `loki.relabel` and `loki.process` and several stage names changed.
>
> **Заметка о Promtail.** Весь 2025 год стандартным агентом Loki был **Promtail**. **2 марта 2026 года** он достиг **End-of-Life** — новых релизов и исправлений безопасности больше нет. Его преемник **Grafana Alloy** объединяет Promtail, агент Prometheus и OpenTelemetry Collector в одном бинарнике. В этой лабе используется Alloy. Конфиги Promtail всё ещё могут встретиться в унаследованных системах, поэтому слайды 14–15 лекции показывают устаревший синтаксис и его соответствие Alloy — но **не поднимай новый Promtail в 2026 году.** Блок `pipeline_stages` в Promtail **не** переносится в Alloy один к одному: эквиваленты разнесены по `loki.relabel` и `loki.process`, а часть названий стадий изменилась.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что у тебя должно быть из предыдущих лаб:**

- `app_python/` from Lab 1 — Flask/FastAPI service with `/` and `/health`
  * `app_python/` из лабы 1 — сервис на Flask/FastAPI с `/` и `/health`
- A working Docker image of it from Lab 2 (registry-pushed or local tag)
  * Рабочий Docker-образ из лабы 2 (запушенный в registry или локальный тег)
- Comfort with Docker Compose v2 from Lab 6 (Jinja2-templated stack)
  * Уверенное владение Docker Compose v2 из лабы 6 (стек с шаблонами Jinja2)

**This lab adds:**

**Эта лаба добавляет:**

- `monitoring/docker-compose.yml` — the Loki + Alloy + Grafana stack
  * `monitoring/docker-compose.yml` — стек Loki + Alloy + Grafana
- `monitoring/loki/config.yml` — provided plumbing
  * `monitoring/loki/config.yml` — готовая обвязка
- `monitoring/alloy/config.alloy` — **YOU write the River pipeline**
  * `monitoring/alloy/config.alloy` — **пайплайн River пишешь ты**
- `monitoring/grafana/dashboards/lab07.json` — your exported 4-panel dashboard
  * `monitoring/grafana/dashboards/lab07.json` — твой экспортированный дашборд из четырёх панелей
- `monitoring/docs/LAB07.md` — submission report
  * `monitoring/docs/LAB07.md` — отчёт по сдаче
- A JSON-logging upgrade to `app_python/app.py`
  * Обновление `app_python/app.py` с JSON-логированием

By Lab 16 you'll redeploy this same idea on Kubernetes as `kube-prometheus-stack` + a Loki Helm chart. The pieces you learn this week (label cardinality, LogQL, ConfigMap-mounted configs) all carry forward.

К лабе 16 ты развернёшь ту же идею в Kubernetes как `kube-prometheus-stack` + Helm-чарт Loki. Всё, что ты изучишь на этой неделе (кардинальность меток, LogQL, конфиги в ConfigMap), пригодится дальше.

---

## Setup

Подготовка

Versions used in this lab — pin these in your compose file:

Версии, используемые в лабе — зафиксируй их в compose-файле:

| Component | Tag | Released |
|---|---|---|
| `grafana/loki` | `3.7.0` | Mar 2026 |
| `grafana/alloy` | `v1.16.1` | May 6 2026 |
| `grafana/grafana` | `13.0.1` | May 12 2026 |

| Компонент | Тег | Дата релиза |
|---|---|---|
| `grafana/loki` | `3.7.0` | март 2026 |
| `grafana/alloy` | `v1.16.1` | 6 мая 2026 |
| `grafana/grafana` | `13.0.1` | 12 мая 2026 |

```bash
docker --version           # 28.x or 29.x
docker compose version     # v2.x — note the space, not a hyphen
curl --version             # for the verification commands below
```

Пояснение: Docker нужен версии 28.x или 29.x; в `docker compose version` важна команда с пробелом, а не дефис; `curl` понадобится для проверок ниже.

Create the directory layout (you'll fill the files yourself):

Создай структуру каталогов (файлы заполнишь сам):

```
monitoring/
├── docker-compose.yml             # YOU write this (§1.4)
├── loki/
│   └── config.yml                 # provided plumbing — copy from labs/lab07/loki/
├── alloy/
│   └── config.alloy               # YOU write this (§1.3)
├── grafana/
│   └── dashboards/
│       └── lab07.json             # exported from Grafana UI in Task 3
└── docs/
    └── LAB07.md                   # your submission report
```

Пояснение: `docker-compose.yml`, `config.alloy` и `lab07.json` пишешь ты (см. §1.4, §1.3 и задание 3); `loki/config.yml` копируется из `labs/lab07/loki/`; `docs/LAB07.md` — итоговый отчёт.

Course-repo plumbing for this lab:

Готовая обвязка из репозитория курса для этой лабы:

- `labs/lab07/loki/config.yml` — drop-in Loki config (see §1.2)
  * `labs/lab07/loki/config.yml` — готовый конфиг Loki (см. §1.2)
- `plumbing/echo/` — optional second log source (see §2.2)
  * `plumbing/echo/` — необязательный второй источник логов (см. §2.2)

---

## Task 1 — Deploy the Loki + Alloy + Grafana stack (3 pts)

Задание 1 — Развернуть стек Loki + Alloy + Grafana (3 балла)

### 1.1 — Read first, write second

1.1 — Сначала прочитай, потом пиши

Read these before you touch a config (they answer the questions the YOUR-TASK blocks ask of you):

Прочитай это, прежде чем трогать конфиги (эти материалы отвечают на вопросы из блоков YOUR TASK):

- [Loki overview](https://grafana.com/docs/loki/latest/get-started/overview/) — distributor / ingester / querier; chunks vs index
  * [Обзор Loki](https://grafana.com/docs/loki/latest/get-started/overview/) — distributor / ingester / querier; чанки и индекс
- [Alloy components](https://grafana.com/docs/alloy/latest/) — `discovery.*`, `loki.source.*`, `loki.write`, how they reference each other by name
  * [Компоненты Alloy](https://grafana.com/docs/alloy/latest/) — `discovery.*`, `loki.source.*`, `loki.write` и то, как они ссылаются друг на друга по имени
- [LogQL intro](https://grafana.com/docs/loki/latest/query/) — selectors, filters, parsers, metric queries
  * [Введение в LogQL](https://grafana.com/docs/loki/latest/query/) — селекторы, фильтры, парсеры, метрические запросы

`YOUR TASK`: in `docs/LAB07.md`, answer these three questions in 2–4 sentences each. You'll come back and revise as you build:

`YOUR TASK`: в `docs/LAB07.md` ответь на эти три вопроса по 2–4 предложения на каждый. Вернёшься и перепишешь их по ходу сборки:

1. How does Loki's **label** index differ from Elasticsearch's full-text index, and why is that cheaper at 100 GB/day?
   * Чем индекс **меток** Loki отличается от полнотекстового индекса Elasticsearch и почему при 100 ГБ в день это дешевле?
2. What is a **stream** in Loki, and what happens to memory if you make `user_id` a label on a service with 10 M users?
   * Что такое **поток (stream)** в Loki и что произойдёт с памятью, если сделать `user_id` меткой для сервиса с 10 млн пользователей?
3. Alloy components reference each other by **name** (e.g. `loki.write.default.receiver`). What's the export name, and why is it different from the block name (`loki.write "default"`)?
   * Компоненты Alloy ссылаются друг на друга по **имени** (например, `loki.write.default.receiver`). Что такое имя экспорта и почему оно отличается от имени блока (`loki.write "default"`)?

### 1.2 — Loki config (provided plumbing — do not rewrite)

1.2 — Конфиг Loki (готовая обвязка — не переписывай)

The Loki config is the **one** fully-written file in this lab; it lives in the course repo as plumbing. Copy it into your stack:

Конфиг Loki — **единственный** полностью готовый файл в этой лабе; он лежит в репозитории курса как обвязка. Скопируй его в свой стек:

```bash
cp labs/lab07/loki/config.yml monitoring/loki/config.yml
```

You don't write it — but you **must** be able to explain in your docs:

Ты его не пишешь — но **обязан** уметь объяснить в документации:

- Why `store: tsdb` + `schema: v13` (and what it replaced)
  * Почему `store: tsdb` + `schema: v13` (и что они заменили)
- Why `object_store: filesystem` is wrong for production
  * Почему `object_store: filesystem` не подходит для продакшена
- Why the `compactor` block is non-negotiable when `retention_period` is set
  * Почему блок `compactor` обязателен, когда задан `retention_period`

> ⚠️ **Gotcha to internalise:** `retention_period` with no `compactor` block = logs never delete. Loki swallows the limit silently. This is the #1 reason teams blow past their disk budget. Reference this in your docs §3 ("Configuration choices").
>
> **Подвох, который нужно усвоить:** `retention_period` без блока `compactor` = логи никогда не удаляются. Loki молча проглатывает лимит. Это причина №1, по которой команды выбиваются из бюджета диска. Ссылайся на это в разделе §3 документации («Configuration choices»).

### 1.3 — Alloy River config (YOUR TASK)

1.3 — Конфиг River для Alloy (YOUR TASK)

**File:** `monitoring/alloy/config.alloy`

**Файл:** `monitoring/alloy/config.alloy`

Alloy is wired components, not a YAML pipeline. Four blocks, each feeding the next. The block names below are non-negotiable — but the arguments, labels, and references between them are the skill.

Alloy — это связанные компоненты, а не YAML-пайплайн. Четыре блока, каждый питает следующий. Имена блоков ниже менять нельзя — но аргументы, метки и ссылки между ними и есть навык, который ты осваиваешь.

`YOUR TASK`: fill the blanks. The block names tell you the role each block plays; the comments tell you the contract.

`YOUR TASK`: заполни пропуски. Имена блоков подсказывают роль каждого блока, а комментарии — контракт.

```alloy
// 1) Discover Docker containers via the daemon socket
discovery.docker "containers" {
  host             = ___           // YOUR TASK: docker socket URL
  refresh_interval = "5s"
  ___                              // YOUR TASK: add a `filter { ... }` block — only
                                   //   scrape containers labelled `logging=alloy`
}

// 2) Promote useful Docker metadata to Loki labels — keep cardinality LOW
discovery.relabel "containers" {
  targets = ___                    // YOUR TASK: previous block's exported targets
  rule {
    source_labels = [___]          // YOUR TASK: Docker meta-label for container name
    regex         = ___            // YOUR TASK: strip the leading "/"
    target_label  = "container"
  }
  ___                              // YOUR TASK: second `rule { ... }` — copy Docker
                                   //   label `app` into Loki label `app`
}

// 3) Tail the discovered containers' log streams
loki.source.docker "default" {
  host       = ___                 // YOUR TASK: same socket as block 1
  targets    = ___                 // YOUR TASK: feed from the *relabel* block
  forward_to = [___]               // YOUR TASK: which loki.write receiver?
}

// 4) Ship everything to Loki
loki.write "default" {
  endpoint {
    url = ___                      // YOUR TASK: Loki push URL — which host, port, path?
  }
}
```

Пояснение: заполни все места `___`; комментарии внутри блоков объясняют, что именно должно оказаться на месте пропуска. Имена блоков (`discovery.docker "containers"`, `discovery.relabel "containers"`, `loki.source.docker "default"`, `loki.write "default"`) менять нельзя.

Notes on each block (don't skip these — they're the *why* behind the blanks):

Заметки по каждому блоку (не пропускай — именно они объясняют, *почему* нужны пропуски):

- **Block 1** — `filter` is how you make logging opt-in. Without it, Alloy ships every container's stdout, including its own and Loki's, polluting your dashboards.
  * **Блок 1** — `filter` делает сбор логов опциональным. Без него Alloy отправляет stdout каждого контейнера, включая свой и Loki, и засоряет дашборды.
- **Block 2** — `discovery.relabel` rewrites label metadata before logs are scraped. The first rule normalises the container name (Docker prefixes it with `/`); the second copies a user-defined Docker label into a Loki label. **Stop and think about cardinality** before adding any third rule — anything with more than ~50 distinct values doesn't belong here.
  * **Блок 2** — `discovery.relabel` переписывает метаданные меток до сбора логов. Первое правило нормализует имя контейнера (Docker добавляет к нему префикс `/`); второе копирует пользовательскую Docker-метку в метку Loki. **Остановись и подумай о кардинальности**, прежде чем добавлять третье правило — всё, что даёт больше ~50 различных значений, здесь не место.
- **Block 3** — `loki.source.docker` tails container stdout/stderr. Its `targets` should come from the **relabel** output (not the raw discovery), otherwise your labels never make it into Loki.
  * **Блок 3** — `loki.source.docker` читает stdout/stderr контейнеров. Его `targets` должны приходить из выхода **relabel** (а не из сырого discovery), иначе твои метки так и не попадут в Loki.
- **Block 4** — `loki.write` is the only block that talks HTTP. Its endpoint URL determines whether your stack is portable across networks; use the compose service name, not `localhost`.
  * **Блок 4** — `loki.write` — единственный блок, который общается по HTTP. От URL эндпоинта зависит, будет ли стек переносимым между сетями; используй имя сервиса из compose, а не `localhost`.

<details>
<summary>💡 River-syntax hints</summary>

Подсказки по синтаксису River

- A block is `<kind>.<subkind> "label" { … }`. The **label** in quotes (e.g. `"containers"`, `"default"`) is how other blocks refer to it.
  * Блок записывается как `<kind>.<subkind> "label" { … }`. **Метка (label)** в кавычках (например, `"containers"`, `"default"`) — это то, по чему на блок ссылаются другие блоки.
- Each component has documented **exports**. `discovery.docker` exports `.targets`. `discovery.relabel` exports `.output`. `loki.write` exports `.receiver`. You wire blocks by writing the fully-qualified export — e.g. `discovery.relabel.containers.output`.
  * У каждого компонента есть документированные **экспорты**. `discovery.docker` экспортирует `.targets`, `discovery.relabel` — `.output`, `loki.write` — `.receiver`. Связывание блоков — это запись полного имени экспорта, например `discovery.relabel.containers.output`.
- Alloy serves a live UI on port **12345** showing the component graph. If a wire is broken, you'll see it red in the UI before you'll see it in Loki.
  * Alloy показывает живую схему компонентов в UI на порту **12345**. Если связь нарушена, ты увидишь это красным в UI раньше, чем в самом Loki.
- Reference pages: [`discovery.docker`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/), [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/), [`loki.source.docker`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/), [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/).
  * Справочные страницы: [`discovery.docker`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/), [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/), [`loki.source.docker`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/), [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/).
- The Loki push endpoint path is documented [here](https://grafana.com/docs/loki/latest/reference/loki-http-api/#ingest-logs).
  * Путь эндпоинта приёма логов Loki описан [здесь](https://grafana.com/docs/loki/latest/reference/loki-http-api/#ingest-logs).

</details>

### 1.4 — Docker Compose stack (YOUR TASK)

1.4 — Стек Docker Compose (YOUR TASK)

**File:** `monitoring/docker-compose.yml`

**Файл:** `monitoring/docker-compose.yml`

The stack has **four** services. The image tags + ports are given (so versions are pinned); the volumes, network membership, env vars, and `depends_on` are your job — those are the wires that make it actually work.

В стеке **четыре** сервиса. Теги образов и порты даны (версии зафиксированы); тома, состав сети, переменные окружения и `depends_on` — твоя работа: именно эти связи заставляют всё работать.

| Service | Image | Published ports | Why it's there |
|---|---|---|---|
| `loki` | `grafana/loki:3.7.0` | `3100:3100` | Storage + query engine |
| `alloy` | `grafana/alloy:v1.16.1` | `12345:12345` (live UI) | Collector — reads docker.sock, pushes to Loki |
| `grafana` | `grafana/grafana:13.0.1` | `3000:3000` | Dashboards + Explore UI |
| `app` | your Lab 2 image | `8080:8080` | Source of logs — Lab 1 service with JSON output (Task 2) |

| Сервис | Образ | Опубликованные порты | Зачем он нужен |
|---|---|---|---|
| `loki` | `grafana/loki:3.7.0` | `3100:3100` | Хранилище и движок запросов |
| `alloy` | `grafana/alloy:v1.16.1` | `12345:12345` (живой UI) | Коллектор — читает docker.sock, отправляет в Loki |
| `grafana` | `grafana/grafana:13.0.1` | `3000:3000` | Дашборды и UI Explore |
| `app` | твой образ из лабы 2 | `8080:8080` | Источник логов — сервис из лабы 1 с JSON-выводом (задание 2) |

`YOUR TASK`: write the compose file. Specifically you must figure out:

`YOUR TASK`: напиши compose-файл. В частности, тебе нужно разобраться:

1. **Volumes for `loki`:** one bind mount makes the config file from §1.2 visible inside the container at `/etc/loki/config.yml`; one named volume persists `/loki` so chunks survive a restart. The container's command should be `-config.file=/etc/loki/config.yml`.
   * **Тома для `loki`:** один bind mount делает конфиг из §1.2 видимым внутри контейнера по пути `/etc/loki/config.yml`; один именованный том сохраняет `/loki`, чтобы чанки пережили перезапуск. Команда контейнера должна быть `-config.file=/etc/loki/config.yml`.
2. **Volumes for `alloy`:** one bind mount for the config from §1.3 (mounted **read-only**); one **read-only** mount of the host Docker socket so Alloy can discover and tail containers. Alloy's command needs three flags: `run`, `--server.http.listen-addr=0.0.0.0:12345`, `--storage.path=/var/lib/alloy/data`, then the config path.
   * **Тома для `alloy`:** один bind mount для конфига из §1.3 (монтируется **только для чтения**); ещё один **read-only**-mount Docker-сокета хоста, чтобы Alloy мог находить контейнеры и читать их логи. Команде Alloy нужны три флага: `run`, `--server.http.listen-addr=0.0.0.0:12345`, `--storage.path=/var/lib/alloy/data`, а затем путь к конфигу.
3. **Env vars for `grafana`:** for this task only, enable anonymous admin access (`GF_AUTH_ANONYMOUS_ENABLED`, `GF_AUTH_ANONYMOUS_ORG_ROLE`). **You will turn this off in Task 4** — leave a comment now so you remember. Persist `/var/lib/grafana` in a named volume so your dashboard from Task 3 survives `docker compose down`.
   * **Переменные окружения для `grafana`:** только для этого задания включи анонимный доступ с правами администратора (`GF_AUTH_ANONYMOUS_ENABLED`, `GF_AUTH_ANONYMOUS_ORG_ROLE`). **В задании 4 ты это отключишь** — оставь комментарий сейчас, чтобы не забыть. Сохрани `/var/lib/grafana` в именованном томе, чтобы дашборд из задания 3 пережил `docker compose down`.
4. **Labels on `app`:** the container must carry `logging: "alloy"` (opts into Alloy collection because of your `filter` in §1.3) and `app: "devops-python"` (becomes a Loki label via your relabel rule in §1.3). Without **both** of those labels, your dashboards in Task 3 will be empty.
   * **Метки на `app`:** контейнер должен нести `logging: "alloy"` (подключает его к сбору в Alloy благодаря твоему `filter` из §1.3) и `app: "devops-python"` (становится меткой Loki через твоё правило relabel из §1.3). Без **обеих** этих меток дашборды в задании 3 будут пустыми.
5. **Network:** define a user-defined bridge network (call it `logging`) and put **all four** services on it so they resolve each other by service name. `loki` and `grafana` must wait for their dependencies before starting (`depends_on:`).
   * **Сеть:** определи пользовательскую bridge-сеть (назови её `logging`) и подключи к ней **все четыре** сервиса, чтобы они находили друг друга по имени сервиса. `loki` и `grafana` должны дождаться зависимостей перед запуском (`depends_on:`).

```yaml
# monitoring/docker-compose.yml — YOUR TASK
services:
  loki:
    image: grafana/loki:3.7.0
    command: ___                              # YOUR TASK: -config.file=/etc/loki/config.yml
    ports:
      - "3100:3100"
    volumes:
      - ___                                   # YOUR TASK: bind-mount loki/config.yml
      - ___                                   # YOUR TASK: named volume for /loki
    networks: [___]                           # YOUR TASK

  alloy:
    image: grafana/alloy:v1.16.1
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - --storage.path=/var/lib/alloy/data
      - ___                                   # YOUR TASK: path to mounted config.alloy
    ports:
      - "12345:12345"
    volumes:
      - ___                                   # YOUR TASK: bind-mount config.alloy :ro
      - ___                                   # YOUR TASK: bind-mount the Docker socket :ro
    networks: [___]                           # YOUR TASK: same network as loki
    depends_on: [___]                         # YOUR TASK: must start after which service?

  grafana:
    image: grafana/grafana:13.0.1
    ports:
      - "3000:3000"
    environment:
      - ___                                   # YOUR TASK: enable anonymous access (DEV ONLY — Task 4 turns this off)
      - ___                                   # YOUR TASK: anonymous role = Admin
    volumes:
      - ___                                   # YOUR TASK: persist /var/lib/grafana
    networks: [___]                           # YOUR TASK
    depends_on: [___]                         # YOUR TASK

  app:
    image: ___                                # YOUR TASK: your Lab 2 image (or `build:` block)
    ports:
      - "8080:8080"
    labels:
      ___: ___                                # YOUR TASK: opt into Alloy collection
      ___: ___                                # YOUR TASK: the `app` Loki label (matches §1.3 rule)
    networks: [___]                           # YOUR TASK

volumes:
  ___:                                        # YOUR TASK: declare named volume for loki
  ___:                                        # YOUR TASK: declare named volume for grafana

networks:
  ___:                                        # YOUR TASK: declare the user-defined bridge
    name: logging
```

> ⚠️ **Security note:** mounting `/var/run/docker.sock` gives Alloy a **powerful** view of the host — effectively root on the daemon. Fine for this lab; in production you'd use a socket proxy (Tecnativa's `docker-socket-proxy`) or file-based discovery. Note this trade-off in your docs.
>
> **Замечание по безопасности:** монтирование `/var/run/docker.sock` даёт Alloy **мощный** доступ к хосту — фактически root на демоне. Для этой лабы это нормально; в продакшене использовали бы socket proxy (например, `docker-socket-proxy` от Tecnativa) или file-based discovery. Отметь этот компромисс в документации.

### 1.5 — Bring it up and verify

1.5 — Запусти и проверь

```bash
cd monitoring
docker compose up -d
docker compose ps
```

Verify:

Проверь:

```bash
# Loki — ingester takes ~15s after process start to report ready.
# If you see "Ingester not ready" the first time, wait and retry; do NOT debug.
curl -s http://localhost:3100/ready          # expect: "ready"

# Loki's view of discovered labels — once Alloy is wired correctly, you should
# see at LEAST "container" and (after §2.2) "app" in the list:
curl -s http://localhost:3100/loki/api/v1/labels | jq -c .data
# (illustrative — your set will differ)
# ["container","service_name"]

# Alloy live component graph — open in your browser
#   macOS:  open http://localhost:12345
#   Linux:  xdg-open http://localhost:12345   (or just paste the URL)

# Grafana — open in your browser at http://localhost:3000
# Add data source: Connections → Data sources → Loki → URL http://loki:3100 → Save & Test
```

Пояснение: `curl .../ready` должен вернуть `ready`; в списке меток из `.../labels` должны появиться минимум `container`, а после §2.2 — `app`; граф компонентов Alloy открывается на `http://localhost:12345`; Grafana — на `http://localhost:3000`, где нужно добавить источник данных Loki с URL `http://loki:3100` и проверить кнопкой Save & Test.

### 1.6 — Proof of work

1.6 — Подтверждение работы

**Paste into `docs/LAB07.md`:**

**Вставь в `docs/LAB07.md`:**

- `docker compose ps` output showing all four services `Up` (after Task 4, loki + grafana must additionally report `(healthy)`; alloy + app are not required to define healthchecks)
  * Вывод `docker compose ps`, где все четыре сервиса в состоянии `Up` (после задания 4 loki и grafana дополнительно должны показывать `(healthy)`; для alloy и app проверки здоровья задавать не обязательно)
- `curl -s http://localhost:3100/ready` output — literally the word `ready`
  * Вывод `curl -s http://localhost:3100/ready` — буквально слово `ready`
- `curl -s http://localhost:3100/loki/api/v1/labels | jq -c .data` — must include `container` (and `app` after §2.2)
  * `curl -s http://localhost:3100/loki/api/v1/labels | jq -c .data` — должен содержать `container` (и `app` после §2.2)
- Screenshot of the Alloy live UI at `:12345` showing the four blocks wired with no red edges
  * Скриншот живого UI Alloy на `:12345`, где четыре блока связаны и нет красных рёбер
- Screenshot of Grafana **Explore** running `{container=~".+"}` returning logs from at least two containers
  * Скриншот Grafana **Explore** с запросом `{container=~".+"}`, возвращающим логи минимум из двух контейнеров

---

## Task 2 — Integrate your app & ship JSON logs (3 pts)

Задание 2 — Подключи своё приложение и отправь JSON-логи (3 балла)

### 2.1 — Add JSON logging to your Lab 1 Python app

2.1 — Добавь JSON-логирование в Python-приложение из лабы 1

Pick **one** library — both are fine; both produce JSON Loki can parse with `| json`.

Выбери **одну** библиотеку — подойдут обе, и обе дают JSON, который Loki разберёт через `| json`.

| Library | When to pick | Trade-off |
|---|---|---|
| **`python-json-logger`** | Drop-in over `logging.basicConfig`. Minimal app changes. | Stdlib `logging`'s mental model — handlers, formatters, levels. |
| **`structlog`** | New code where you want context vars / processors / typed events. | Different API; richer ergonomics. |

| Библиотека | Когда выбирать | Компромисс |
|---|---|---|
| **`python-json-logger`** | Замена поверх `logging.basicConfig`. Минимум изменений в приложении. | Привычная модель стандартного `logging` — handlers, formatters, уровни. |
| **`structlog`** | Новый код, где нужны контекстные переменные, процессоры и типизированные события. | Другой API; богаче эргономика. |

`YOUR TASK`: upgrade `app_python/app.py` so that the following events emit one **JSON line per event** to stdout (which is what Docker captures and Alloy ships):

`YOUR TASK`: обнови `app_python/app.py` так, чтобы следующие события выводили по **одной JSON-строке на событие** в stdout (именно его захватывает Docker и отправляет Alloy):

| Event | When | Required JSON fields |
|---|---|---|
| **Startup** | At process start | `level`, `msg`, `host`, `port`, `service` |
| **Every HTTP request** | After response | `level`, `msg`, `method`, `path`, `status`, `client_ip`, `duration_ms` |
| **Every error** | In a 500 handler or `try/except` | `level="ERROR"`, `msg`, `error_type`, the exception's string |

| Событие | Когда | Обязательные поля JSON |
|---|---|---|
| **Запуск (Startup)** | При старте процесса | `level`, `msg`, `host`, `port`, `service` |
| **Каждый HTTP-запрос** | После ответа | `level`, `msg`, `method`, `path`, `status`, `client_ip`, `duration_ms` |
| **Каждая ошибка** | В обработчике 500 или в `try/except` | `level="ERROR"`, `msg`, `error_type`, строка исключения |

Hints:

Подсказки:

- Flask: a `@app.before_request` to capture start time + `@app.after_request` to log; or use `werkzeug`'s built-in logger and replace its formatter with a JSON one.
  * Flask: `@app.before_request` для захвата времени старта + `@app.after_request` для логирования; или встроенный логгер `werkzeug` с заменой форматтера на JSON.
- FastAPI: a single middleware with `time.perf_counter()` brackets is cleaner than per-route logging.
  * FastAPI: одно middleware с замерами через `time.perf_counter()` чище, чем логирование в каждом маршруте.
- **Do not** put `request_id` or `user_id` into a **label**. Put them in the JSON body — you'll filter on them at query time with `| json | user_id="alice"`. (Lecture 7, slide 12.)
  * **Не** помещай `request_id` или `user_id` в **метку**. клади их в тело JSON — фильтровать будешь при запросе через `| json | user_id="alice"`. (Лекция 7, слайд 12.)
- One JSON object per line. No trailing commas. Don't pretty-print — `docker logs` will then split your log lines on every internal newline and your dashboards will be a mess.
  * Один JSON-объект на строку. Без запятых в конце. Не форматируй красиво — иначе `docker logs` разобьёт твои строки лога на каждом внутреннем переводе строки, и дашборды превратятся в кашу.

Update `app_python/requirements.txt`:

Обнови `app_python/requirements.txt`:

```
# python-json-logger>=3.4.0    # YOUR TASK: pin one of these to an exact version
# structlog>=25.4.0
```

Пояснение: раскомментируй одну из библиотек и зафиксируй точную версию.

Rebuild your image (Lab 2's Dockerfile) so the new dependency lands in the container.

Пересобери образ (Dockerfile из лабы 2), чтобы новая зависимость попала в контейнер.

### 2.2 — Add your app to the stack

2.2 — Добавь своё приложение в стек

You already declared the `app` service in §1.4. Two labels are non-negotiable:

Сервис `app` ты уже объявил в §1.4. Две метки обязательны:

```yaml
    labels:
      logging: "alloy"          # opts the container into Alloy collection (§1.3 filter)
      app: "devops-python"      # becomes a Loki label via the §1.3 relabel rule
```

Пояснение: `logging: "alloy"` включает контейнер в сбор Alloy (благодаря `filter` из §1.3); `app: "devops-python"` превращается в метку Loki через правило relabel из §1.3.

**Optional but recommended — `plumbing/echo`.** The course repo ships a tiny Go service at `plumbing/echo` (you'll see it again in Lab 9). Add it as a second log source so your dashboards aren't single-app — same two labels, port `8081`, image or `build:` line of your choice.

**Необязательно, но рекомендуется — `plumbing/echo`.** В репозитории курса есть крошечный сервис на Go в `plumbing/echo` (ты снова встретишь его в лабе 9). Добавь его вторым источником логов, чтобы дашборды не были про одно приложение, — те же две метки, порт `8081`, образ или строка `build:` на твой выбор.

### 2.3 — Generate traffic and query

2.3 — Сгенерируй трафик и выполни запросы

```bash
# Generate at least 20 successful + 5 not-found + 1 error event
for i in $(seq 1 20); do curl -s http://localhost:8080/ > /dev/null; done
for i in $(seq 1 5);  do curl -s http://localhost:8080/nope > /dev/null; done
# Force an error: hit an endpoint that raises (add a /boom route or trip a 500 deliberately)
curl -s http://localhost:8080/boom > /dev/null
```

### 2.4 — Write the LogQL queries (YOUR TASK)

2.4 — Напиши запросы LogQL (YOUR TASK)

For each row below, write the LogQL that answers it. Don't copy from the lecture — answer the question, then verify by running it in Explore.

Для каждой строки таблицы ниже напиши LogQL, отвечающий на вопрос. Не копируй из лекции — сначала ответь на вопрос, затем проверь, выполнив запрос в Explore.

| # | Question | Hint |
|---|---|---|
| Q1 | All logs from your Python app | Stream selector on the `app` label, nothing else |
| Q2 | Only the JSON lines where `level` is `ERROR` | Stream selector → `\| json` → field equality on `level` |
| Q3 | Rate of errors per minute, per container | Metric query: `sum by (container) (rate(... [1m]))` — chain `\| json \| level="ERROR"` after the stream selector, then wrap the whole log expression in `rate(...[1m])` |
| Q4 | All requests to a path containing `health` (filter on the **JSON field `path`**, not a substring of the raw line) | `\| json` then a field-equality / regex filter on `path` |

| # | Вопрос | Подсказка |
|---|---|---|
| Q1 | Все логи твоего Python-приложения | Селектор потока по метке `app`, больше ничего |
| Q2 | Только JSON-строки, где `level` равен `ERROR` | Селектор потока → `\| json` → проверка равенства поля `level` |
| Q3 | Частота ошибок в минуту по каждому контейнеру | Метрический запрос: `sum by (container) (rate(... [1m]))` — прицепи `\| json \| level="ERROR"` после селектора потока, затем оберни всё лог-выражение в `rate(...[1m])` |
| Q4 | Все запросы к пути, содержащему `health` (фильтр по **JSON-полю `path`**, а не по подстроке сырой строки) | `\| json`, затем фильтр равенства или regex по `path` |

> 💡 **Performance order:** stream selector → line filter → parser → field filter → metric. The cheapest filter goes first. Filters cascade left-to-right.
>
> **Порядок производительности:** селектор потока → фильтр строк → парсер → фильтр полей → метрика. Самый дешёвый фильтр идёт первым. Фильтры применяются слева направо.

### 2.5 — Proof of work

2.5 — Подтверждение работы

**Paste into `docs/LAB07.md`:**

**Вставь в `docs/LAB07.md`:**

- One full JSON log line from `docker logs <app-container> | head -n 1` — must show all the required fields from §2.1
  * Одна полная JSON-строка из `docker logs <app-container> | head -n 1` — она должна содержать все обязательные поля из §2.1
- `curl -s http://localhost:3100/loki/api/v1/label/container/values | jq -c .data` — must include your app container's name
  * `curl -s http://localhost:3100/loki/api/v1/label/container/values | jq -c .data` — должен содержать имя контейнера твоего приложения
- The output of running Q1–Q4 through the Loki HTTP query API (so the grader sees the data, not just a screenshot):
  * Вывод выполнения Q1–Q4 через HTTP API запросов Loki (чтобы проверяющий увидел данные, а не только скриншот):
  ```bash
  curl --get http://localhost:3100/loki/api/v1/query_range \
       --data-urlencode 'query=<YOUR QUERY HERE>' \
       --data-urlencode 'limit=5' | jq '.data.result[0].values[0]'
  ```
  Show each of Q1–Q4 returning at least one of *your own app's* log lines. (Q3 returns a numeric series, not a log line — show `.data.result[0]` for that one.)

  Покажи, что каждый из Q1–Q4 возвращает хотя бы одну строку лога *твоего собственного приложения*. (Q3 возвращает числовой ряд, а не строку лога — для него покажи `.data.result[0]`.)

- The four LogQL queries themselves, in a code block, with one sentence each of what they do
  * Сами четыре запроса LogQL в блоке кода, с одним предложением о том, что делает каждый

---

## Task 3 — Build a log dashboard (2 pts)

Задание 3 — Построй дашборд по логам (2 балла)

### 3.1 — One question per panel

3.1 — Один вопрос на панель

A dashboard answers **one question** for **one audience**. The four panels below cover the most common operational questions for a single service. The LogQL queries from Task 2 already cover most of what you need; one is new.

Дашборд отвечает на **один вопрос** для **одной аудитории**. Четыре панели ниже покрывают самые частые операционные вопросы по одному сервису. Запросы LogQL из задания 2 уже покрывают почти всё нужное; один запрос новый.

| Panel | Visualisation | Question it answers | Source query |
|---|---|---|---|
| 1 | Logs | *"What's happening right now?"* | Your Q1 (broaden to `app=~"devops-.*"` if you have multiple apps) |
| 2 | Time series | *"How busy are we, per app?"* | Request **rate** per app (write this — same shape as Q3 but no error filter) |
| 3 | Logs | *"What broke?"* | Your Q2 |
| 4 | Pie chart or Stat | *"How noisy is each level?"* | Distribution of `level` over 5 min — use `count_over_time` + `| json` + `sum by (level)` |

| Панель | Визуализация | На какой вопрос отвечает | Исходный запрос |
|---|---|---|---|
| 1 | Логи | *«Что происходит прямо сейчас?»* | Твой Q1 (расширь до `app=~"devops-.*"`, если приложений несколько) |
| 2 | Временной ряд | *«Насколько мы загружены, по каждому приложению?»* | **Частота** запросов по приложению (напиши сам — та же форма, что у Q3, но без фильтра ошибок) |
| 3 | Логи | *«Что сломалось?»* | Твой Q2 |
| 4 | Круговая диаграмма или Stat | *«Насколько шумно на каждом уровне?»* | Распределение `level` за 5 минут — используй `count_over_time` + `| json` + `sum by (level)` |

`YOUR TASK`: write the LogQL for panels 2 and 4 yourself (don't copy from §2.4). Panels 1 and 3 reuse Q1/Q2 verbatim.

`YOUR TASK`: напиши LogQL для панелей 2 и 4 сам (не копируй из §2.4). Панели 1 и 3 используют Q1 и Q2 без изменений.

### 3.2 — Build it

3.2 — Собери дашборд

1. **Dashboards → New → New dashboard → Add visualization**. Pick the **Loki** data source.
   * **Dashboards → New → New dashboard → Add visualization**. Выбери источник данных **Loki**.
2. Paste the LogQL. Choose the visualisation type from the table above.
   * Вставь LogQL. Выбери тип визуализации из таблицы выше.
3. Title each panel after the **question it answers**, not the data source. ("Errors per minute" beats "Loki query #3".)
   * Называй панель по **вопросу, на который она отвечает**, а не по источнику данных. («Ошибок в минуту» лучше, чем «Запрос Loki #3».)
4. Save the dashboard. Then **Share → Export → Save to file** and commit the JSON model to `monitoring/grafana/dashboards/lab07.json`.
   * Сохрани дашборд. Затем **Share → Export → Save to file** и закоммить JSON-модель в `monitoring/grafana/dashboards/lab07.json`.

> 💡 If a panel is empty, the cause is almost always missing labels, not bad LogQL. Run the panel's query in **Explore** first; if it works there but not in the panel, it's a time-range or data-source-default issue.
>
> Если панель пустая, причина почти всегда в отсутствующих метках, а не в плохом LogQL. Сначала выполни запрос панели в **Explore**; если там он работает, а в панели нет — дело в диапазоне времени или источнике данных по умолчанию.

### 3.3 — Proof of work

3.3 — Подтверждение работы

**Paste into `docs/LAB07.md`:**

**Вставь в `docs/LAB07.md`:**

- Screenshot of the 4-panel dashboard showing real data on all four panels (generate fresh traffic right before the screenshot — empty panels = no credit)
  * Скриншот дашборда из четырёх панелей с реальными данными на всех четырёх (сгенерируй свежий трафик прямо перед скриншотом — пустые панели не засчитываются)
- The four LogQL queries in a markdown table with the question each answers
  * Четыре запроса LogQL в markdown-таблице с вопросом, на который отвечает каждый
- The committed JSON file path: `monitoring/grafana/dashboards/lab07.json`
  * Путь закоммиченного JSON-файла: `monitoring/grafana/dashboards/lab07.json`

---

## Task 4 — Production readiness (1 pt)

Задание 4 — Готовность к продакшену (1 балл)

Harden the stack so it isn't a toy.

Укрепи стек, чтобы он не оставался игрушкой.

### 4.1 — Resource limits

4.1 — Лимиты ресурсов

`YOUR TASK`: add `deploy.resources.limits` (cpus + memory) and `reservations` to each of the four services. Pick sane numbers — Loki and Grafana need more memory than Alloy and the app; the app needs almost nothing. Document your choices in `LAB07.md`.

`YOUR TASK`: добавь `deploy.resources.limits` (cpus + memory) и `reservations` каждому из четырёх сервисов. Выбери разумные значения — Loki и Grafana требуют больше памяти, чем Alloy и приложение; приложению почти ничего не нужно. Опиши свой выбор в `LAB07.md`.

### 4.2 — Secure Grafana

4.2 — Защити Grafana

`YOUR TASK`:
- Set `GF_AUTH_ANONYMOUS_ENABLED=false` (your DEV-ONLY comment in §1.4 reminded you).
  * Установи `GF_AUTH_ANONYMOUS_ENABLED=false` (комментарий DEV-ONLY из §1.4 напомнит).
- Set the admin password from a `.env` file: `GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}`.
  * Задай пароль администратора из файла `.env`: `GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}`.
- Add `.env` to **`.gitignore`** — and commit the `.gitignore` change.
  * Добавь `.env` в **`.gitignore`** — и закоммить изменение `.gitignore`.
- Commit a `.env.example` with the variable name and an empty placeholder so the grader knows the contract.
  * Закоммить `.env.example` с именем переменной и пустым значением-заглушкой, чтобы проверяющий знал контракт.

### 4.3 — Health checks

4.3 — Проверки здоровья

`YOUR TASK`: add a `healthcheck:` block to **Loki** (test against `:3100/ready`) and **Grafana** (test against `:3000/api/health`). Use `wget --spider` or `curl --fail` inside the container. Pick reasonable `interval`, `timeout`, `retries`, and `start_period` — Loki specifically takes ~15s to be ready after start, so `start_period` must give it that grace.

`YOUR TASK`: добавь блок `healthcheck:` для **Loki** (проверка `:3100/ready`) и **Grafana** (проверка `:3000/api/health`). Используй `wget --spider` или `curl --fail` внутри контейнера. Выбери разумные `interval`, `timeout`, `retries` и `start_period` — Loki после старта становится готов примерно через 15 секунд, так что `start_period` должен дать ему этот запас.

### 4.4 — Proof of work

4.4 — Подтверждение работы

**Paste into `docs/LAB07.md`:**

**Вставь в `docs/LAB07.md`:**

- `docker compose ps` output where Loki and Grafana both report `(healthy)`
  * Вывод `docker compose ps`, где и Loki, и Grafana показывают `(healthy)`
- Screenshot of the Grafana **login page** (i.e. anonymous access is off)
  * Скриншот **страницы входа** Grafana (то есть анонимный доступ выключен)
- Listing of `monitoring/.env.example` and confirmation that `.env` is in `.gitignore` and **not** committed (a `git log -- monitoring/.env` returning nothing is fine evidence)
  * Содержимое `monitoring/.env.example` и подтверждение, что `.env` есть в `.gitignore` и **не** закоммичен (пустой вывод `git log -- monitoring/.env` — достаточное доказательство)

---

## Task 5 — Documentation (1 pt)

Задание 5 — Документация (1 балл)

`YOUR TASK`: write `monitoring/docs/LAB07.md` with these sections, in order:

`YOUR TASK`: напиши `monitoring/docs/LAB07.md` со следующими разделами по порядку:

1. **Architecture** — a Mermaid diagram of app → Alloy → Loki → Grafana, with the network and the Docker socket marked
   * **Архитектура** — диаграмма Mermaid: app → Alloy → Loki → Grafana, с отмеченными сетью и Docker-сокетом
2. **Setup** — how to deploy (`docker compose up -d`) and the verification commands from §1.5
   * **Установка** — как развернуть (`docker compose up -d`) и команды проверки из §1.5
3. **Configuration choices** — your three answers from §1.1, plus a short note on each Loki block (schema, retention, compactor) and each Alloy component
   * **Выбор конфигурации** — твои три ответа из §1.1 плюс короткая заметка по каждому блоку Loki (schema, retention, compactor) и каждому компоненту Alloy
4. **Application logging** — which library you picked + why + the field schema you emit
   * **Логирование приложения** — какую библиотеку выбрал, почему и какую схему полей выводишь
5. **Dashboard** — each panel, its LogQL, the question it answers, and a screenshot
   * **Дашборд** — каждая панель, её LogQL, вопрос, на который она отвечает, и скриншот
6. **Production config** — your resource limits + reasoning, Grafana auth, the Docker-socket trade-off
   * **Продовая конфигурация** — лимиты ресурсов и обоснование, аутентификация Grafana, компромисс с Docker-сокетом
7. **Challenges & solutions** — at least one real one (not "I was new to Loki")
   * **Трудности и решения** — хотя бы одна настоящая (не «я был новичком в Loki»)

Include config **snippets** (not whole files) and the captures from Tasks 1–4. Keep it readable; this is the artefact your future on-call self will read at 3 am.

Вставляй **фрагменты** конфигов (а не целые файлы) и подтверждения из заданий 1–4. Пиши читаемо — это артефакт, который твоё будущее дежурное «я» будет читать в три часа ночи.

---

## Bonus Task — Ansible automation (2 pts)

Бонусное задание — Автоматизация через Ansible (2 балла)

Automate the whole stack with Ansible, building on Lab 6.

Автоматизируй весь стек через Ansible, опираясь на лабу 6.

`YOUR TASK`: create `roles/monitoring` that brings the entire stack up idempotently.

`YOUR TASK`: создай `roles/monitoring`, которая идемпотентно поднимает весь стек.

The role must:

Роль должна:

- **Template** `loki/config.yml`, `alloy/config.alloy`, and `docker-compose.yml` from Jinja2 — image tags, ports, retention, and resource limits become role variables (defined in `defaults/main.yml`)
  * **Шаблонизировать** `loki/config.yml`, `alloy/config.alloy` и `docker-compose.yml` через Jinja2 — теги образов, порты, ретеншн и лимиты ресурсов становятся переменными роли (объявлены в `defaults/main.yml`)
- **Deploy** with `community.docker.docker_compose_v2`
  * **Разворачивать** через `community.docker.docker_compose_v2`
- **Wait** for Loki `:3100/ready` *and* Grafana `:3000/api/health` before reporting success (use `ansible.builtin.uri` with retries)
  * **Дожидаться** готовности Loki `:3100/ready` *и* Grafana `:3000/api/health`, прежде чем сообщить об успехе (используй `ansible.builtin.uri` с повторами)
- Be **idempotent**: a second run reports `changed=0` (Lab 5's headline criterion still applies)
  * Быть **идемпотентной**: повторный запуск даёт `changed=0` (главный критерий лабы 5 по-прежнему действует)

Less hand-holding than Task 1–5: figure out the directory layout, the variable names, and the readiness polling pattern yourself. Lab 6 covered the `docker_compose_v2` deploy mechanics.

Подсказок меньше, чем в заданиях 1–5: структуру каталогов, имена переменных и схему опроса готовности продумай сам. Механику развёртывания через `docker_compose_v2` разбирали в лабе 6.

**Evidence (paste into `docs/LAB07.md`):**

**Доказательства (вставь в `docs/LAB07.md`):**

- First-run output (changes > 0) and second-run output (`changed=0`)
  * Вывод первого запуска (изменения > 0) и вывод второго (`changed=0`)
- The **rendered** (not template) `docker-compose.yml` from a real run
  * **Отрендеренный** (не шаблон) `docker-compose.yml` из реального запуска
- Path: `ansible/roles/monitoring/` + `ansible/playbooks/deploy-monitoring.yml`
  * Путь: `ansible/roles/monitoring/` + `ansible/playbooks/deploy-monitoring.yml`

---

## How to Submit

Как сдавать

```bash
git switch -c lab07
git add monitoring/
git add app_python/                            # JSON logging upgrade
git add ansible/roles/monitoring \
        ansible/playbooks/deploy-monitoring.yml   # only if bonus done
git commit -m "feat(lab07): loki + alloy + grafana observability stack"
git push -u origin lab07
```

Пояснение: команда создаёт ветку `lab07`, добавляет в неё каталог `monitoring/`, обновлённый `app_python/` и (если сделан бонус) файлы Ansible, затем коммитит и пушит.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab07` → `course-repo:master` *(reviewed)*
  * `your-fork:lab07` → `course-repo:master` *(с ревью)*
- `your-fork:lab07` → `your-fork:master` *(merges into your own main)*
  * `your-fork:lab07` → `your-fork:master` *(сливается в твой собственный main)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — stack up, Alloy pipeline wired, container labels visible
- [ ] Task 2 done — JSON logging in app_python, 4 LogQL queries verified via /query_range
- [ ] Task 3 done — 4-panel dashboard built, JSON exported into the repo
- [ ] Task 4 done — limits, healthchecks, anonymous off, .env gitignored
- [ ] Task 5 done — LAB07.md with all 7 sections + captures + screenshots
- [ ] Bonus done — idempotent Ansible role with readiness wait
```

Пояснение: чек-лист подтверждает, что задание 1 (стек, пайплайн Alloy, метки), задание 2 (JSON-логи, четыре запроса LogQL), задание 3 (дашборд из четырёх панелей), задание 4 (лимиты, healthchecks, отключённый анонимный доступ, `.env` в `.gitignore`), задание 5 (`LAB07.md` с семью разделами) и бонус (идемпотентная роль Ansible) выполнены.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Stack deployment (3 pts)

Задание 1 — Развёртывание стека (3 балла)

- ✅ `docker compose up -d` brings up loki, alloy, grafana, app
  * ✅ `docker compose up -d` поднимает loki, alloy, grafana, app
- ✅ `curl -s :3100/ready` returns `ready`
  * ✅ `curl -s :3100/ready` возвращает `ready`
- ✅ `curl -s :3100/loki/api/v1/labels` includes `container` (and `app` after §2.2)
  * ✅ `curl -s :3100/loki/api/v1/labels` содержит `container` (и `app` после §2.2)
- ✅ Alloy live UI shows the four-block pipeline with no broken wires
  * ✅ Живой UI Alloy показывает пайплайн из четырёх блоков без разорванных связей
- ✅ Grafana Explore `{container=~".+"}` returns logs from ≥ 2 containers
  * ✅ Grafana Explore `{container=~".+"}` возвращает логи минимум из двух контейнеров

### Task 2 — App integration + JSON logging (3 pts)

Задание 2 — Интеграция приложения и JSON-логирование (3 балла)

- ✅ App emits one JSON object per line; required fields present per event type
  * ✅ Приложение выводит один JSON-объект на строку; обязательные поля присутствуют для каждого типа события
- ✅ App container is labelled `logging=alloy` and `app=devops-python`
  * ✅ У контейнера приложения есть метки `logging=alloy` и `app=devops-python`
- ✅ Q1–Q4 each return real data via `/loki/api/v1/query_range` (CLI captures pasted)
  * ✅ Каждый из Q1–Q4 возвращает реальные данные через `/loki/api/v1/query_range` (вывод CLI вставлен)
- ✅ Q3 / Q4 use the JSON parser + field filters (not raw-line regex)
  * ✅ Q3 и Q4 используют JSON-парсер и фильтры по полям (а не regex по сырой строке)

### Task 3 — Dashboard (2 pts)

Задание 3 — Дашборд (2 балла)

- ✅ 4 panels, one per operational question, all showing live data
  * ✅ 4 панели, по одной на операционный вопрос, все показывают живые данные
- ✅ Panel titles describe the question, not the query
  * ✅ Заголовки панелей описывают вопрос, а не запрос
- ✅ Dashboard JSON committed to `monitoring/grafana/dashboards/lab07.json`
  * ✅ JSON дашборда закоммичен в `monitoring/grafana/dashboards/lab07.json`

### Task 4 — Production config (1 pt)

Задание 4 — Продовая конфигурация (1 балл)

- ✅ All four services have CPU + memory limits and reservations
  * ✅ У всех четырёх сервисов заданы лимиты и reservations по CPU и памяти
- ✅ Grafana anonymous OFF; admin password from `.env`; `.env` gitignored
  * ✅ Анонимный доступ Grafana выключен; пароль администратора из `.env`; `.env` в `.gitignore`
- ✅ Loki and Grafana report `(healthy)` in `docker compose ps`
  * ✅ Loki и Grafana показывают `(healthy)` в `docker compose ps`

### Task 5 — Documentation (1 pt)

Задание 5 — Документация (1 балл)

- ✅ All seven sections present in `monitoring/docs/LAB07.md`
  * ✅ Все семь разделов присутствуют в `monitoring/docs/LAB07.md`
- ✅ Research answers from §1.1 are in your own words
  * ✅ Ответы на вопросы из §1.1 изложены своими словами
- ✅ Real screenshots + CLI captures (not placeholders)
  * ✅ Настоящие скриншоты и вывод CLI (не заглушки)

### Bonus — Ansible (2 pts)

Бонус — Ansible (2 балла)

- ✅ Role templates all three configs from variables
  * ✅ Роль шаблонизирует все три конфига через переменные
- ✅ Deploy is idempotent — second run `changed=0`
  * ✅ Развёртывание идемпотентно — повторный запуск даёт `changed=0`
- ✅ Readiness wait blocks success until Loki + Grafana respond
  * ✅ Ожидание готовности не даёт отчитаться об успехе, пока Loki и Grafana не ответят
- ✅ Both playbook runs captured in docs
  * ✅ Оба запуска плейбука зафиксированы в документации

---

## Rubric

Рубрика оценивания

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Stack deployment | **3** | All four services up; Alloy pipeline correctly wired (the four YOUR-TASKs in §1.3 + the volumes/network/labels in §1.4); `/ready` + `/labels` proofs |
| **Task 2** — App integration | **3** | JSON logging implemented; required fields; Q1–Q4 each return real data via the HTTP API |
| **Task 3** — Dashboard | **2** | 4 panels with appropriate LogQL; dashboard JSON committed |
| **Task 4** — Production config | **1** | Limits, anonymous off, healthchecks healthy |
| **Task 5** — Docs | **1** | All seven sections, real captures, research answers in your own words |
| **Bonus** — Ansible | **2** | Idempotent templated deployment with readiness wait |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Развёртывание стека | **3** | Все четыре сервиса работают; пайплайн Alloy связан верно (четыре YOUR TASK из §1.3 + тома, сеть и метки из §1.4); доказательства `/ready` и `/labels` |
| **Задание 2** — Интеграция приложения | **3** | JSON-логирование реализовано; обязательные поля на месте; Q1–Q4 возвращают реальные данные через HTTP API |
| **Задание 3** — Дашборд | **2** | 4 панели с подходящим LogQL; JSON дашборда закоммичен |
| **Задание 4** — Продовая конфигурация | **1** | Лимиты, выключенный анонимный доступ, рабочие healthchecks |
| **Задание 5** — Документация | **1** | Все семь разделов, настоящие подтверждения, ответы на вопросы своими словами |
| **Бонус** — Ansible | **2** | Идемпотентное шаблонизированное развёртывание с ожиданием готовности |
| **Итого** | **12** | 10 основных + 2 бонусных |

---

## Resources

Ресурсы

<details>
<summary>📚 Loki</summary>

- [Loki overview](https://grafana.com/docs/loki/latest/get-started/overview/)
  * [Обзор Loki](https://grafana.com/docs/loki/latest/get-started/overview/)
- [Loki configuration reference](https://grafana.com/docs/loki/latest/configure/)
  * [Справочник по конфигурации Loki](https://grafana.com/docs/loki/latest/configure/)
- [TSDB storage](https://grafana.com/docs/loki/latest/operations/storage/tsdb/)
  * [Хранилище TSDB](https://grafana.com/docs/loki/latest/operations/storage/tsdb/)
- [Retention & compactor](https://grafana.com/docs/loki/latest/operations/storage/retention/)
  * [Ретеншн и compactor](https://grafana.com/docs/loki/latest/operations/storage/retention/)
- [HTTP API — push & query](https://grafana.com/docs/loki/latest/reference/loki-http-api/)
  * [HTTP API — отправка и запросы](https://grafana.com/docs/loki/latest/reference/loki-http-api/)

</details>

<details>
<summary>⚡ Grafana Alloy (Promtail's successor)</summary>

Grafana Alloy (преемник Promtail)

- [Alloy documentation](https://grafana.com/docs/alloy/latest/)
  * [Документация Alloy](https://grafana.com/docs/alloy/latest/)
- [`discovery.docker`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/)
- [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/)
- [`loki.source.docker`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/)
- [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/)
- [Migrate from Promtail to Alloy](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/)
  * [Переход с Promtail на Alloy](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/)

</details>

<details>
<summary>🔍 LogQL & Grafana</summary>

LogQL и Grafana

- [LogQL reference](https://grafana.com/docs/loki/latest/query/) — every operator with examples
  * [Справочник по LogQL](https://grafana.com/docs/loki/latest/query/) — каждый оператор с примерами
- [Grafana dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
  * [Дашборды Grafana](https://grafana.com/docs/grafana/latest/dashboards/)
- [Loki data source](https://grafana.com/docs/grafana/latest/datasources/loki/)
  * [Источник данных Loki](https://grafana.com/docs/grafana/latest/datasources/loki/)
- [Explore logs](https://grafana.com/docs/grafana/latest/explore/logs-integration/)
  * [Логи в Explore](https://grafana.com/docs/grafana/latest/explore/logs-integration/)

</details>

<details>
<summary>📝 Structured logging</summary>

Структурированное логирование

- [`structlog`](https://www.structlog.org/en/stable/)
- [`python-json-logger`](https://github.com/nhairs/python-json-logger)
- [The Twelve-Factor App: Logs](https://12factor.net/logs)
  * [The Twelve-Factor App: логи](https://12factor.net/logs)
- [Python `logging` HOWTO](https://docs.python.org/3/howto/logging.html)
  * [Руководство по `logging` в Python](https://docs.python.org/3/howto/logging.html)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

Частые ошибки (из реальных прогонов)

- **"Ingester not ready, waiting 15s" on first push.** Loki's ingester takes ~15s after the process is alive to report ready. If you query immediately after `docker compose up -d` and see empty results, that's not a bug — it's startup. Wait, retry. Set `healthcheck.start_period: 30s` in Task 4 so compose stops claiming the service is healthy before it actually is.
  * **«Ingester not ready, waiting 15s» при первой отправке.** Ingester Loki становится готовым примерно через 15 секунд после запуска процесса. Если сразу после `docker compose up -d` запрос возвращает пустые результаты, это не баг, а запуск. Подожди и повтори. В задании 4 задай `healthcheck.start_period: 30s`, чтобы compose не объявлял сервис здоровым раньше времени.
- **High-cardinality labels = Loki OOM.** Never put `user_id`, `request_id`, `trace_id`, `email`, or any per-request value into a Loki label (Alloy `discovery.relabel` rule or otherwise). Each unique combination is a stream, each stream costs index + memory, and at 10 M users you'll crash the ingester. Per-request values go in the **JSON body** and you filter at query time with `| json | user_id="alice"`.
  * **Метки с высокой кардинальностью = OOM в Loki.** Никогда не помещай `user_id`, `request_id`, `trace_id`, `email` или любое значение, уникальное для запроса, в метку Loki (в правиле `discovery.relabel` в Alloy или как-то иначе). Каждая уникальная комбинация — это поток, каждый поток стоит индекса и памяти, и при 10 млн пользователей ты уронишь ingester. Значения уровня запроса клади в **тело JSON** и фильтруй при запросе через `| json | user_id="alice"`.
- **Docker socket permissions inside the Alloy container.** On a default Linux install, `/var/run/docker.sock` is owned by root and the `docker` group; inside the Alloy container, the runtime user may not be in that group. If Alloy logs `permission denied` reading the socket, either run the container with `user: root` for this lab (acceptable for dev) or pass `group_add: ["${DOCKER_GID}"]` after capturing the host's docker GID with `getent group docker | cut -d: -f3`.
  * **Права на Docker-сокет внутри контейнера Alloy.** В стандартной установке Linux `/var/run/docker.sock` принадлежит root и группе `docker`; внутри контейнера Alloy пользователь процесса может не входить в эту группу. Если Alloy пишет `permission denied` при чтении сокета, либо запусти контейнер с `user: root` для этой лабы (для разработки допустимо), либо передай `group_add: ["${DOCKER_GID}"]`, предварительно узнав GID docker на хосте командой `getent group docker | cut -d: -f3`.
- **`subPath` mounts don't refresh on ConfigMap changes** — you'll meet this hard wall in Lab 12 when you redo Loki on Kubernetes. For now, a Compose bind-mount of `loki/config.yml` *does* update when you `docker compose restart loki`, so changes are easy. Internalise the behaviour now; you'll need it later.
  * **Монтирования через `subPath` не обновляются при изменении ConfigMap** — с этой стеной ты столкнёшься в лабе 12, когда будешь переделывать Loki в Kubernetes. Пока же bind-mount `loki/config.yml` в Compose *обновляется* при `docker compose restart loki`, так что менять конфиг легко. Запомни это поведение сейчас — позже оно понадобится.
- **Promtail config syntax does NOT translate 1:1 to Alloy.** Promtail's flat list of `pipeline_stages` is split in Alloy between `loki.relabel` (label rewrites) and `loki.process` (parsing / line manipulation). Stage names changed (`docker:` → `stage.docker {}` block, `json:` → `stage.json {}`, etc.). If you have a Promtail config to migrate, use the [official migration tool](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/) rather than translating by hand — the field rename matrix is wider than it looks.
  * **Синтаксис конфига Promtail НЕ переносится в Alloy один к одному.** Плоский список `pipeline_stages` в Promtail разделяется в Alloy между `loki.relabel` (переписывание меток) и `loki.process` (парсинг и работа со строками). Названия стадий изменились (`docker:` → блок `stage.docker {}`, `json:` → `stage.json {}` и т. д.). Если тебе нужно мигрировать конфиг Promtail, используй [официальный инструмент миграции](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/), а не переводи вручную — матрица переименований полей шире, чем кажется.
- **Returning a dict from FastAPI's middleware vs Flask's `after_request`.** FastAPI middleware is awaited; Flask hooks aren't. Don't `await` `request.json()` inside a Flask after_request — it's blocking. Pick one framework's pattern and stay consistent.
  * **Возврат dict из middleware FastAPI и из `after_request` во Flask.** Middleware FastAPI вызывается с await, хуки Flask — нет. Не делай `await` у `request.json()` внутри Flask after_request — это блокирующая операция. Выбери паттерн одного фреймворка и придерживайся его.
- **JSON log line split by `docker logs`.** If you pretty-print JSON (`json.dumps(obj, indent=2)`), `docker logs` (and therefore Alloy) treats each indented line as a separate log entry — and your `| json` parser will fail on every one of them. Always one JSON object per line, no indentation.
  * **Строка JSON-лога, разбитая `docker logs`.** Если выводить JSON красиво (`json.dumps(obj, indent=2)`), `docker logs` (а значит, и Alloy) считает каждую строку с отступом отдельной записью лога — и твой парсер `| json` упадёт на каждой из них. Всегда один JSON-объект на строку, без отступов.
- **Anonymous Grafana left enabled in Task 4 "by accident".** You said "I'll remember to turn it off" in §1.4. You won't — the comment in your compose file is the only thing that will save you. Leave it.
  * **Анонимный доступ в Grafana, забытый в задании 4 «случайно».** В §1.4 ты сказал: «Я не забуду выключить». Не выключишь — спасти может только комментарий в compose-файле. Оставь его.
- **Port 8080 collides with Lab 6.** Lab 6's Compose stack defaults to host port 8080 too. If you didn't `docker compose down` it before starting this lab, `docker compose up -d` errors with *"Bind for 0.0.0.0:8080 failed: port is already allocated"*. Either bring Lab 6 down (`cd ../app_python && docker compose down`), or pick a different host port for Lab 6's stack (`app_host_port: 8084` in your Lab 6 group_vars). The container labels — `logging=alloy` and `app=devops-python` — must live on **this** lab's `app` container regardless.
  * **Порт 8080 конфликтует с лабой 6.** Стек Compose из лабы 6 по умолчанию тоже занимает хост-порт 8080. Если не выполнить `docker compose down` перед началом этой лабы, `docker compose up -d` упадёт с ошибкой *«Bind for 0.0.0.0:8080 failed: port is already allocated»*. Либо останови лабу 6 (`cd ../app_python && docker compose down`), либо выбери другой хост-порт для стека лабы 6 (`app_host_port: 8084` в group_vars лабы 6). Метки контейнера — `logging=alloy` и `app=devops-python` — в любом случае должны быть на контейнере `app` **этой** лабы.
- **Loki's `/ready` returns 503 before `(healthy)` shows up.** The healthcheck in Task 4 polls `/ready`; until Loki's ingester comes up (~15s after process start), `/ready` 503s and `docker compose ps` reports `(starting)`. Set `start_period: 30s` so compose doesn't flap to `(unhealthy)` then back to `(healthy)` and confuse you about whether the stack works.
  * **`/ready` в Loki возвращает 503 до появления `(healthy)`.** Healthcheck из задания 4 опрашивает `/ready`; пока ingester Loki не поднимется (~15 секунд после старта процесса), `/ready` отдаёт 503, а `docker compose ps` показывает `(starting)`. Задай `start_period: 30s`, чтобы compose не метался между `(unhealthy)` и `(healthy)` и не путал тебя насчёт работоспособности стека.

</details>

<details>
<summary>🛠️ Dev tools worth knowing</summary>

Инструменты разработчика, которые стоит знать

- [`logcli`](https://grafana.com/docs/loki/latest/query/logcli/) — a CLI for LogQL; faster than the Grafana UI for one-shot queries
  * [`logcli`](https://grafana.com/docs/loki/latest/query/logcli/) — CLI для LogQL; быстрее UI Grafana для разовых запросов
- [`jq`](https://jqlang.github.io/jq/) — JSON pretty-printer; chain `curl … | jq` everywhere
  * [`jq`](https://jqlang.github.io/jq/) — форматер JSON; везде цепляй `curl … | jq`
- [`hey`](https://github.com/rakyll/hey) — generate traffic against your app to populate dashboards
  * [`hey`](https://github.com/rakyll/hey) — генератор трафика к твоему приложению, чтобы наполнять дашборды

</details>

---

## Looking Ahead

Что дальше

| Lab | What it adds |
|---:|---|
| 8 | The **metrics** pillar — Prometheus + `/metrics` on your Lab 1 app, RED-method PromQL |
| 9 | Deploy this same app + the `echo` plumbing service on k3d Kubernetes |
| 12 | ConfigMaps + PVCs — re-mount the Loki config from a ConfigMap; meet the `subPath` foot-gun |
| 16 | `kube-prometheus-stack` + a Loki Helm chart — the K8s redo of this whole stack |

| Лаба | Что добавляет |
|---:|---|
| 8 | Столп **метрик** — Prometheus + `/metrics` в приложении из лабы 1, PromQL по методу RED |
| 9 | Развернуть это же приложение + сервис-обвязку `echo` в Kubernetes на k3d |
| 12 | ConfigMaps + PVC — смонтировать конфиг Loki из ConfigMap; познакомиться с подводным камнем `subPath` |
| 16 | `kube-prometheus-stack` + Helm-чарт Loki — переделка всего этого стека для Kubernetes |

---

**Good luck!** 🚀

**Удачи!** 🚀

> **Remember:** Loki indexes *labels*, not text. Keep label cardinality low; per-request fields belong in the JSON body, filtered at query time with `| json | field="value"`. The Alloy pipeline is four wired components — not a YAML list — and the wires are export names (`.targets`, `.output`, `.receiver`), not block labels.
>
> **Запомни:** Loki индексирует *метки*, а не текст. Держи кардинальность меток низкой; поля уровня запроса клади в тело JSON и фильтруй их при запросе через `| json | field="value"`. Пайплайн Alloy — это четыре связанных компонента, а не список YAML, и связи задаются именами экспортов (`.targets`, `.output`, `.receiver`), а не метками блоков.
