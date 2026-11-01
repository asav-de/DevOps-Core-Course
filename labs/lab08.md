# Lab 8 — Metrics & Monitoring with Prometheus

Лаба 8 — Метрики и мониторинг с Prometheus

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Metrics%20%26%20Monitoring-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Prometheus%203.11%20|%20Grafana%2013%20|%20prometheus--client-informational)

> **Goal:** instrument your Lab 1 Python app with Prometheus metrics, deploy Prometheus 3.x next to last week's Loki stack to scrape them, and design a Grafana dashboard around the **RED method**.
> **Deliverable:** a PR from `lab08` adding the `/metrics` endpoint + instrumentation to `app_python/`, a `prometheus` service in `monitoring/docker-compose.yml`, `monitoring/prometheus/prometheus.yml`, a 6-panel RED dashboard JSON, and `monitoring/docs/LAB08.md` with the evidence captures.
>
> **Цель:** инструментировать Python-приложение из лабы 1 метриками Prometheus, развернуть Prometheus 3.x рядом со стеком Loki из прошлой недели, чтобы он собирал эти метрики, и спроектировать дашборд Grafana вокруг **метода RED**.
> **Что сдать:** PR из ветки `lab08`, который добавляет эндпоинт `/metrics` и инструментирование в `app_python/`, сервис `prometheus` в `monitoring/docker-compose.yml`, файл `monitoring/prometheus/prometheus.yml`, JSON 6-панельного RED-дашборда и `monitoring/docs/LAB08.md` с доказательствами.

---

## Overview

Обзор

In Lab 7 you gave your service eyes (logs). This week you give it a **pulse** (metrics). You'll:

В лабе 7 ты дал своему сервису глаза (логи). На этой неделе ты даёшь ему **пульс** (метрики). Ты:

- Instrument an HTTP service with `prometheus_client` — declaring a **Counter**, a **Histogram**, and a `/metrics` endpoint
  * Инструментировать HTTP-сервис через `prometheus_client` — объявить **Counter**, **Histogram** и эндпоинт `/metrics`
- Wire **Prometheus** into the Lab 7 Compose stack so it pulls `/metrics` from your app every few seconds
  * Подключить **Prometheus** к стеку Compose из лабы 7, чтобы он каждые несколько секунд забирал `/metrics` у приложения
- Write **PromQL** for the **RED method** (Rate / Errors / Duration) — you write the queries, not copy them
  * Написать **PromQL** для **метода RED** (Rate / Errors / Duration) — запросы пишешь ты, а не копируешь
- Design a Grafana dashboard around the three RED questions plus saturation
  * Спроектировать дашборд Grafana вокруг трёх вопросов RED плюс saturation (насыщение)
- Harden the stack: retention, resource limits, health checks, persistence
  * Укрепить стек: retention, лимиты ресурсов, health checks, персистентность

> ⚠️ **Scope:** single-binary Prometheus on the local Docker host. No Alertmanager, no `remote_write`, no Mimir — those are post-Core territory. The skill is **instrumentation + PromQL**, not running federated Prometheus.
>
> ⚠️ **Границы темы:** один экземпляр Prometheus на локальном Docker-хосте. Без Alertmanager, без `remote_write`, без Mimir — это уже за пределами Core. Здесь навык — **инструментирование + PromQL**, а не эксплуатация федеративного Prometheus.

> ✨ **Prometheus 3.x note.** This lab uses Prometheus 3.11.3 (current stable May 2026; 3.5 LTS is supported through Jul 2026). Versus 2.x it defaults to UTF-8 metric/label names, ships an OTLP receiver, and promotes **native histograms to GA**. You'll use classic histograms in the required tasks and may opt into native histograms as a stretch.
>
> ✨ **Замечание о Prometheus 3.x.** В лабе используется Prometheus 3.11.3 (актуальный стабильный релиз на май 2026; LTS 3.5 поддерживается до июля 2026). По сравнению с 2.x он по умолчанию использует имена метрик и меток в UTF-8, поставляется с OTLP-приёмником и переводит **нативные гистограммы в GA**. В обязательных заданиях ты будешь использовать классические гистограммы, а нативные можно взять как дополнительную цель.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**У тебя должно остаться от прошлых лаб:**

- `app_python/` from Lab 1 — Flask/FastAPI service with `/` and `/health`
  * `app_python/` из лабы 1 — сервис на Flask/FastAPI с `/` и `/health`
- A working Docker image of it from Lab 2
  * Рабочий Docker-образ из лабы 2
- The `monitoring/` compose stack from Lab 7 — `loki`, `alloy`, `grafana`, `app`, all on a `logging` network
  * Стек Compose в `monitoring/` из лабы 7 — `loki`, `alloy`, `grafana`, `app`, все в сети `logging`
- The JSON-logging upgrade to `app_python/` from Lab 7 (this lab adds *another* layer to the same `app.py`)
  * Обновление JSON-логирования в `app_python/` из лабы 7 (эта лаба добавляет *ещё один* слой в тот же `app.py`)

**This lab adds:**

**Эта лаба добавляет:**

- A `/metrics` endpoint + `Counter` + `Histogram` instrumentation in `app_python/app.py`
  * Эндпоинт `/metrics` + инструментирование `Counter` и `Histogram` в `app_python/app.py`
- `monitoring/prometheus/prometheus.yml` — YOU fill it from the scaffold
  * `monitoring/prometheus/prometheus.yml` — ТЫ заполняешь его из каркаса
- A `prometheus` service in `monitoring/docker-compose.yml` (extends Lab 7, doesn't replace it)
  * Сервис `prometheus` в `monitoring/docker-compose.yml` (расширяет лабу 7, а не заменяет её)
- `monitoring/grafana/dashboards/lab08.json` — your exported 6-panel RED dashboard
  * `monitoring/grafana/dashboards/lab08.json` — твой экспортированный 6-панельный RED-дашборд
- `monitoring/docs/LAB08.md` — submission report
  * `monitoring/docs/LAB08.md` — отчёт о сдаче

By Lab 16 you'll redeploy this as `kube-prometheus-stack` on k3d with `ServiceMonitor` CRDs auto-discovering targets. Everything you learn this week — metric types, cardinality, PromQL, RED — carries forward unchanged.

К лабе 16 ты переразвернёшь всё это как `kube-prometheus-stack` на k3d, где CRD `ServiceMonitor` сами находят цели. Всё, что ты изучишь на этой неделе — типы метрик, кардинальность, PromQL, RED, — останется актуальным без изменений.

---

## Setup

Подготовка

Versions used in this lab — pin these in your compose file and `requirements.txt`:

Версии, используемые в лабе, — зафиксируй их в compose-файле и `requirements.txt`:

| Component | Tag / Version | Notes |
|---|---|---|
| `prom/prometheus` | `v3.11.3` | Current stable (May 2026); `v3.5.0` LTS is also acceptable |
| `grafana/grafana` | `13.0.1` | Already in your Lab 7 stack |
| `prometheus-client` (Python) | `0.23.1` | `0.23+` minimum; `0.25.x` is the latest patch line |

| Компонент | Тег / Версия | Примечания |
|---|---|---|
| `prom/prometheus` | `v3.11.3` | Актуальная стабильная версия (май 2026); `v3.5.0` LTS тоже подойдёт |
| `grafana/grafana` | `13.0.1` | Уже есть в стеке лабы 7 |
| `prometheus-client` (Python) | `0.23.1` | Минимум `0.23+`; `0.25.x` — последняя ветка патчей |

```bash
docker --version           # 28.x or 29.x
docker compose version     # v2.x
python --version           # 3.12+
```

Пояснение: проверяем версии Docker, Docker Compose и Python — нужны Docker 28.x или 29.x, Compose v2 и Python 3.12+.

Directory layout you'll build:

Структура каталогов, которую ты построишь:

```
app_python/
├── app.py                            # YOU add instrumentation here (§1)
└── requirements.txt                  # YOU add prometheus-client (§1.2)

monitoring/
├── docker-compose.yml                # YOU extend with a `prometheus` service (§2.2)
├── prometheus/
│   └── prometheus.yml                # YOU fill the scaffold (§2.3)
├── grafana/
│   └── dashboards/
│       └── lab08.json                # exported from Grafana UI in Task 3
└── docs/
    └── LAB08.md                      # your submission report
```

Пояснение: комментарии `#` в дереве показывают, что добавляешь именно ты; `YOUR` — твоя часть работы, `§` — ссылка на раздел лабы.

Course-repo plumbing for this lab:

Служебные файлы курса для этой лабы:

- `labs/lab8/prometheus/prometheus.yml` — **scaffold** with YOUR-TASK markers. Copy it into `monitoring/prometheus/prometheus.yml` and fill the blanks.
  * `labs/lab8/prometheus/prometheus.yml` — **каркас** с маркерами YOUR-TASK. Скопируй его в `monitoring/prometheus/prometheus.yml` и заполни пропуски.

---

## Task 1 — Instrument the Python app (3 pts)

Задание 1 — Инструментирование Python-приложения (3 балла)

### 1.1 — Read first, write second

1.1 — Сначала прочитать, потом писать

Read these before you touch `app.py` — they answer the questions the instrumentation asks of you:

Прочитай это, прежде чем трогать `app.py`, — материалы отвечают на вопросы, которые ставит инструментирование:

- [Prometheus metric types](https://prometheus.io/docs/concepts/metric_types/) — Counter / Gauge / Histogram / Summary
  * [Типы метрик Prometheus](https://prometheus.io/docs/concepts/metric_types/) — Counter / Gauge / Histogram / Summary
- [Instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/) — the "low cardinality" rule and metric naming
  * [Лучшие практики инструментирования](https://prometheus.io/docs/practices/instrumentation/) — правило «низкой кардинальности» и именование метрик
- [`prometheus_client` (Python)](https://github.com/prometheus/client_python) — Counter, Histogram, `generate_latest`, `CONTENT_TYPE_LATEST`
  * [`prometheus_client` (Python)](https://github.com/prometheus/client_python) — Counter, Histogram, `generate_latest`, `CONTENT_TYPE_LATEST`

`YOUR TASK`: in `docs/LAB08.md`, answer these three questions in 2–4 sentences each. You'll come back and revise after Task 3:

`YOUR TASK`: в `docs/LAB08.md` ответь на эти три вопроса по 2–4 предложения на каждый. К ним ты вернёшься и перепишешь после задания 3:

1. When do you reach for a **Counter** vs a **Histogram**? Give one example of each from *your own* app.
   * Когда ты берёшь **Counter**, а когда **Histogram**? Приведи по одному примеру из *своего* приложения.
2. Why do you query `rate(counter[5m])` instead of the counter's raw value? (Hint: counters reset on process restart.)
   * Почему ты запрашиваешь `rate(counter[5m])`, а не сырое значение счётчика? (Подсказка: счётчики сбрасываются при перезапуске процесса.)
3. What is **label cardinality**, and why would `user_id` or `request_id` as a label eventually OOM Prometheus?
   * Что такое **кардинальность меток** и почему `user_id` или `request_id` в качестве метки в конце концов приведут к падению Prometheus из-за нехватки памяти (OOM)?

### 1.2 — Add the client library

1.2 — Добавить клиентскую библиотеку

Add the exact pin to `app_python/requirements.txt`:

Зафиксируй точную версию в `app_python/requirements.txt`:

```txt
prometheus-client==0.23.1
```

Пояснение: фиксируем точную версию библиотеки, чтобы сборка была воспроизводимой.

Rebuild your Lab 2 image so the dependency lands in the container.

Пересобери образ из лабы 2, чтобы зависимость попала в контейнер.

### 1.3 — Define the metrics (`YOUR TASK`)

1.3 — Определить метрики (`YOUR TASK`)

You'll add **two** metrics to `app_python/app.py`. The lecture (slides 8, 10, 13) names them, gives the labels, and shows the SRE-default bucket set. Your job is to declare them, give them sensible **help strings**, and pick the right label set — bounded enumerations only.

Ты добавишь **две** метрики в `app_python/app.py`. Лекция (слайды 8, 10, 13) называет их, даёт метки и показывает набор бакетов SRE по умолчанию. Твоя задача — объявить их, задать осмысленные строки **help** и выбрать правильный набор меток — только ограниченные перечисления.

`YOUR TASK`: declare two module-level metrics:

`YOUR TASK`: объяви две метрики на уровне модуля:

| Variable | Type | Metric name | Labels | Notes |
|---|---|---|---|---|
| `REQUESTS` | `Counter` | `app_requests_total` | `method`, `endpoint`, `status` | Counts every HTTP request your service answers |
| `LATENCY` | `Histogram` | `app_request_duration_seconds` | `method`, `endpoint` | Buckets the request duration. Use the SRE defaults: `(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0)` |

| Переменная | Тип | Имя метрики | Метки | Примечания |
|---|---|---|---|---|
| `REQUESTS` | `Counter` | `app_requests_total` | `method`, `endpoint`, `status` | Считает каждый HTTP-запрос, который обслуживает сервис |
| `LATENCY` | `Histogram` | `app_request_duration_seconds` | `method`, `endpoint` | Распределяет длительность запроса по бакетам. Используй набор SRE по умолчанию: `(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0)` |

```python
# app.py — add to your Lab 1 service (Flask shown; adapt to FastAPI middleware)
import time
from flask import Flask, Response, request
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)

# --- Metric definitions (low-cardinality labels only!) ---
REQUESTS = Counter(
    ___,                # YOUR TASK: metric name — see the table; ends in _total per Prom naming convention
    ___,                # YOUR TASK: help string — read by anyone debugging your metrics
    ___,                # YOUR TASK: label list — bounded enumerations only
)
LATENCY = Histogram(
    ___,                # YOUR TASK: metric name — _seconds suffix is the Prom convention for time
    ___,                # YOUR TASK: help string
    ___,                # YOUR TASK: label list — narrower than the counter's (no status)
    buckets=___,        # YOUR TASK: the SRE-default tuple from the table — bucket past your SLO ceiling
)
```

Пояснение: это каркас объявления метрик; вместо каждого `___` подставь имя метрики, строку help, список меток или кортеж бакетов — подсказки в комментариях и таблице выше.

> ⚠️ **Cardinality is the #1 way to OOM Prometheus.** Each unique label combination is its own time series (~3–5 KB of RAM). Labels must be *bounded* enumerations: `method` (~8), `endpoint` (dozens, normalised), `status` (~10). **Never** put `user_id`, `request_id`, raw paths, emails, or timestamps in a label. Same lesson as Loki (Lec 7), same blast radius.
>
> ⚠️ **Кардинальность — главная причина падения Prometheus из-за нехватки памяти (OOM).** Каждая уникальная комбинация меток — отдельная временная серия (~3–5 КБ ОЗУ). Метки должны быть *ограниченными* перечислениями: `method` (~8), `endpoint` (десятки, нормализованные), `status` (~10). **Никогда** не клади в метку `user_id`, `request_id`, сырые пути, email или временные метки. Тот же урок, что с Loki (лекция 7), тот же масштаб последствий.

### 1.4 — Wire the hooks (`YOUR TASK`)

1.4 — Подключить хуки (`YOUR TASK`)

Flask gives you `before_request` (fires before the handler runs) and `after_request` (fires after — gets the `Response` object). The pattern is *start a timer in before, increment + observe in after*. The bodies are yours to write.

Flask даёт тебе `before_request` (срабатывает до обработчика) и `after_request` (срабатывает после — получает объект `Response`). Шаблон такой: *запустить таймер в before, увеличить счётчик и записать наблюдение в after*. Тела функций пишешь ты.

```python
@app.before_request
def _start_timer():
    request._t0 = ___                # YOUR TASK: capture a monotonic start time
                                     #   — hint: time.perf_counter(), NOT time.time()

@app.after_request
def _record(response):
    duration = ___                   # YOUR TASK: now - request._t0
    if ___:                          # YOUR TASK: skip /metrics — exclude its own path here
        return response              #            (otherwise scrape traffic poisons your counts)
    endpoint = ___                   # YOUR TASK: use request.url_rule.rule when matched,
                                     #            fall back to "unknown" when url_rule is None
    REQUESTS.labels(___).inc()       # YOUR TASK: method, endpoint, str(response.status_code)
    LATENCY.labels(___).observe(___) # YOUR TASK: (method, endpoint) labels + the duration
    return response

@app.route("/metrics")
def metrics():
    return Response(
        ___,                         # YOUR TASK: bytes from generate_latest()
        mimetype=___,                # YOUR TASK: CONTENT_TYPE_LATEST — wrong mimetype = silent
                                     #            parse failure on Prometheus's side
    )
```

Пояснение: заполни `___` — сними монотонное время через `time.perf_counter()`, вычисли длительность, исключи путь `/metrics`, возьми нормализованный эндпоинт из `request.url_rule.rule` и передай метки в `REQUESTS` и `LATENCY`.

Hints — *not* full code:

Подсказки — *не* готовый код:

- **`request.url_rule.rule`** is `"/health"` or `"/"` for matched routes. It's `None` for 404s — fall back to `"unknown"` so cardinality stays bounded.
  * **`request.url_rule.rule`** для совпавших маршрутов равен `"/health"` или `"/"`. Для 404 он равен `None` — подставь `"unknown"`, чтобы кардинальность оставалась ограниченной.
- **Cast the status** to `str(response.status_code)`. A label value must be a string.
  * **Приведи статус** к `str(response.status_code)`. Значение метки должно быть строкой.
- **FastAPI** equivalent: one `@app.middleware("http")` coroutine bracketing `await call_next(request)` with `time.perf_counter()`; expose `/metrics` with `Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)`.
  * **Эквивалент для FastAPI:** одна корутина `@app.middleware("http")`, которая оборачивает `await call_next(request)` и использует `time.perf_counter()`; эндпоинт `/metrics` отдай через `Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)`.

### 1.5 — Test locally

1.5 — Проверить локально

You can test in either of two ways — pick one and be consistent:

Проверить можно двумя способами — выбери один и придерживайся его:

- **In the Lab 7 Compose stack** (recommended): rebuild + restart just the `app` service so the new image + `prometheus-client` are running on `localhost:8080`.
  * **В стеке Compose из лабы 7** (рекомендуется): пересобери и перезапусти только сервис `app`, чтобы новый образ с `prometheus-client` работал на `localhost:8080`.
- **Standalone Python**: `pip install -r requirements.txt` into a venv first, then `python app.py`.
  * **Отдельным Python-процессом:** сначала `pip install -r requirements.txt` в venv, затем `python app.py`.

```bash
# Option A — through the Compose stack (recommended; matches Task 2 below).
# Run from repo root; the stack listens on host port 8080:
docker compose -f monitoring/docker-compose.yml up -d --build app    # picks up new app.py + requirements

# Option B — standalone Python (run from repo root; Lab 1 default port is 5000).
# Either keep the curls below on :5000, or set PORT=8080 to match Option A:
# PORT=8080 python app_python/app.py

# Generate traffic and inspect /metrics (Option A → :8080; Option B unchanged → :5000):
for i in $(seq 1 20); do curl -s localhost:8080/ > /dev/null; done
for i in $(seq 1 5);  do curl -s localhost:8080/health > /dev/null; done
curl -s localhost:8080/metrics | grep -E "^app_(requests_total|request_duration)"
```

Пояснение: вариант A поднимает только сервис `app` через compose; вариант B запускает приложение напрямую. Ниже генерируется трафик, чтобы метрики появились, и выводится содержимое `/metrics`.

The output is the Prometheus text exposition format. **Illustrative — your counts will differ:**

На выходе — текстовый формат экспозиции Prometheus. **Пример — твои числа будут другими:**

```
app_requests_total{endpoint="/",method="GET",status="200"} 20.0
app_requests_total{endpoint="/health",method="GET",status="200"} 5.0
app_request_duration_seconds_bucket{endpoint="/",method="GET",le="0.005"} 12.0
app_request_duration_seconds_bucket{endpoint="/",method="GET",le="0.01"}  18.0
app_request_duration_seconds_count{endpoint="/",method="GET"} 20.0
app_request_duration_seconds_sum{endpoint="/",method="GET"} 0.0421
```

Пояснение: `_bucket` с меткой `le` — накопленные счётчики по границам бакетов, `_count` и `_sum` — общее число наблюдений и суммарная длительность.

**Sanity check:** call `curl -s localhost:8080/metrics > /tmp/m1.txt`, hit a route, `curl -s localhost:8080/metrics > /tmp/m2.txt`, diff. The total should **increase by exactly 1** for the route you hit — and the `endpoint="/metrics"` line should **not** exist. If `/metrics` is counting itself, your exclude in §1.4 is wrong.

**Контрольная проверка:** выполни `curl -s localhost:8080/metrics > /tmp/m1.txt`, дёрни маршрут, затем `curl -s localhost:8080/metrics > /tmp/m2.txt` и сравни. Счётчик должен **увеличиться ровно на 1** для запрошенного маршрута, а строки `endpoint="/metrics"` быть **не должно**. Если `/metrics` считает сам себя, твоё исключение в §1.4 неверно.

### 1.6 — Proof of work

1.6 — Доказательства работы

**Paste into `docs/LAB08.md`:**

**Вставь в `docs/LAB08.md`:**

- Full `curl -s localhost:8080/metrics | grep ^app_` output showing both `app_requests_total{...}` and `app_request_duration_seconds_{bucket,sum,count}` lines, with your real counts.
  * Полный вывод `curl -s localhost:8080/metrics | grep ^app_` с обеими строками — `app_requests_total{...}` и `app_request_duration_seconds_{bucket,sum,count}` — и твоими реальными числами.
- The "no self-counting" proof: `curl -s localhost:8080/metrics | grep -c 'endpoint="/metrics"'` returning `0`.
  * Доказательство «счётчик не считает себя»: `curl -s localhost:8080/metrics | grep -c 'endpoint="/metrics"'` возвращает `0`.
- The metric-definition + hook code (snippet, not the whole file) committed to `app_python/app.py`.
  * Код объявления метрик и хуков (фрагмент, не весь файл), закоммиченный в `app_python/app.py`.
- Your three written answers from §1.1.
  * Твои три письменных ответа из §1.1.

---

## Task 2 — Deploy Prometheus & wire the scrape (3 pts)

Задание 2 — Развернуть Prometheus и настроить сбор метрик (3 балла)

### 2.1 — Understand the pull model

2.1 — Понять модель pull

`YOUR TASK`: be able to answer these in `docs/LAB08.md` §3 (write the answers now, revise after §2.4):

`YOUR TASK`: будь готов ответить на это в `docs/LAB08.md` §3 (напиши ответы сейчас, пересмотри после §2.4):

1. Why does a *failed scrape* give you target health "for free" (`up == 0`), where a push model can't?
   * Почему *неудачный сбор* даёт тебе состояние цели «бесплатно» (`up == 0`), а модель push — нет?
2. What's the difference between a **job**, a **target**, and the **scrape interval**?
   * В чём разница между **job**, **target** и **scrape interval**?
3. In Docker Compose, why must Prometheus scrape `app:8080` (service name) and **not** `localhost:8080`?
   * В Docker Compose почему Prometheus должен собирать метрики с `app:8080` (имя сервиса), а **не** с `localhost:8080`?

### 2.2 — Add the Prometheus service (`YOUR TASK`)

2.2 — Добавить сервис Prometheus (`YOUR TASK`)

**File:** `monitoring/docker-compose.yml` (extend the Lab 7 stack — don't replace it)

**Файл:** `monitoring/docker-compose.yml` (расширь стек лабы 7 — не заменяй его)

Prometheus joins the same `logging` network so it can reach Loki, Grafana, and your `app` by service name. The image + ports are given (so versions are pinned); the volumes, command flags, healthcheck, and network membership are your job.

Prometheus подключается к той же сети `logging`, чтобы обращаться к Loki, Grafana и твоему `app` по имени сервиса. Образ и порты даны (чтобы версии были зафиксированы); тома, флаги команды, healthcheck и принадлежность к сети — твоя работа.

```yaml
  # add to monitoring/docker-compose.yml, alongside loki/alloy/grafana/app from Lab 7
  prometheus:
    image: prom/prometheus:v3.11.3
    command:
      - ___                              # YOUR TASK: --config.file= path INSIDE the container
                                         #   (where you'll mount prometheus.yml below)
      - ___                              # YOUR TASK: --storage.tsdb.retention.time=
                                         #   (pick a value — Task 4 documents the trade-off)
      - ___                              # YOUR TASK: --storage.tsdb.retention.size=
                                         #   (defence in depth against disk fill)
    ports:
      - "9090:9090"
    volumes:
      - ___                              # YOUR TASK: bind-mount prometheus.yml :ro
      - ___                              # YOUR TASK: named volume for /prometheus (TSDB)
    networks: [___]                      # YOUR TASK: which network from Lab 7?
    healthcheck:
      test: ___                          # YOUR TASK: probe /-/healthy via wget --spider or curl --fail
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

volumes:
  prometheus-data:                       # add alongside loki-data / grafana-data from Lab 7
```

Пояснение: заполни `___` путём к конфигу внутри контейнера, флагами retention по времени и размеру, bind-mount `prometheus.yml` только для чтения, именованным томом для TSDB, сетью `logging` и командой проверки `/-/healthy`.

Notes — don't skip these, they're the *why* behind the blanks:

Замечания — не пропускай их, в них *причина* пустых мест:

- **`--config.file=`** must point at where you mounted `prometheus.yml` (typically `/etc/prometheus/prometheus.yml`). Mismatched paths give a clean "open ...: no such file" line in `docker compose logs prometheus` on startup — read the logs first when it won't come up.
  * **`--config.file=`** должен указывать туда, куда ты смонтировал `prometheus.yml` (обычно `/etc/prometheus/prometheus.yml`). При несовпадении путей в `docker compose logs prometheus` при старте будет чистая строка «open ...: no such file» — сначала читай логи, если сервис не поднимается.
- **Retention is a Prometheus CLI flag, not a config-file setting** (this is the most-Googled Prometheus 3.x footgun). `retention.time` (e.g. `15d`) AND `retention.size` (e.g. `2GB`) — whichever triggers first wins. Document your choices in §4.
  * **Retention — это флаг CLI Prometheus, а не настройка в конфиг-файле** (самая гуглимая ошибка при переходе на Prometheus 3.x). `retention.time` (например, `15d`) И `retention.size` (например, `2GB`) — срабатывает то, что наступит раньше. Задокументируй свой выбор в §4.
- **TSDB volume:** without persistence, every `docker compose down` wipes your metrics history. Mount a named volume at `/prometheus` so dashboards don't go blank after a restart.
  * **Том TSDB:** без персистентности каждый `docker compose down` стирает историю метрик. Смонтируй именованный том в `/prometheus`, чтобы дашборды не пустели после перезапуска.
- **`logging` network** is the one from Lab 7. Putting Prometheus on a different network means it can't resolve `app` / `loki` / `grafana` — and you'd be debugging DNS instead of scraping.
  * **Сеть `logging`** — та же, что в лабе 7. Если поставить Prometheus в другую сеть, он не сможет разрешить имена `app` / `loki` / `grafana`, и ты будешь отлаживать DNS вместо сбора метрик.

### 2.3 — Fill the scrape config (`YOUR TASK`)

2.3 — Заполнить конфиг сбора (`YOUR TASK`)

**File:** `monitoring/prometheus/prometheus.yml`

**Файл:** `monitoring/prometheus/prometheus.yml`

The structure (one `global` block, one `scrape_configs` list, one job per target) is non-negotiable Prometheus. The skill is **what cadence to scrape at**, **which hostname:port each target lives on**, and **what to name each job** (because the name shows up in every dashboard label and every alert).

Структура (один блок `global`, один список `scrape_configs`, один job на цель) — это обязательная основа Prometheus. Навык в том, **с какой периодичностью собирать**, **на каком hostname:port живёт каждая цель** и **как назвать каждый job** (имя попадает в метки каждого дашборда и каждого алерта).

Start from the scaffold:

Начни с каркаса:

```bash
cp labs/lab8/prometheus/prometheus.yml monitoring/prometheus/prometheus.yml
```

Пояснение: копируем каркас из репозитория курса в каталог стека мониторинга.

What you're filling in (the scaffold has the same blanks):

Что ты заполняешь (в каркасе те же пропуски):

```yaml
global:
  scrape_interval:        # YOUR TASK: how often to scrape
                          #   — recall lecture §15: query ranges must be ≥ 4× this
  evaluation_interval:    # YOUR TASK: how often to evaluate rules (match scrape_interval)

scrape_configs:
  - job_name:             # YOUR TASK: short lowercase name, e.g. "prometheus"
    static_configs:
      - targets:          # YOUR TASK: Prometheus's own /metrics, from INSIDE the container
                          #   — this is the ONE target that uses localhost

  - job_name:             # YOUR TASK: a name you'll filter on later — e.g. up{job="..."}
    static_configs:
      - targets:          # YOUR TASK: your app's Compose service name + port
                          #   — NOT localhost; Prometheus runs in a different container

  - job_name:             # YOUR TASK
    static_configs:
      - targets:          # YOUR TASK: Loki's HTTP port on the logging network

  - job_name:             # YOUR TASK
    static_configs:
      - targets:          # YOUR TASK: Grafana's HTTP port on the logging network
```

Пояснение: заполни интервал сбора, интервал вычисления правил и четыре job-а — сам Prometheus, приложение, Loki и Grafana; для каждого укажи `targets`.

Notes:

Замечания:

- `metrics_path:` defaults to `/metrics`, so omit it.
  * `metrics_path:` по умолчанию равен `/metrics`, поэтому его можно не указывать.
- **Service names are hostnames** on a Compose user-defined network — `loki:3100`, `grafana:3000`, `app:8080`. The Prometheus self-scrape is the only one using `localhost` because that traffic stays inside the Prometheus container.
  * **Имена сервисов — это hostname** в пользовательской сети Compose: `loki:3100`, `grafana:3000`, `app:8080`. Единственная цель, для которой законно используется `localhost`, — сбор метрик самим Prometheus, потому что этот трафик не покидает его контейнер.
- **Choose `scrape_interval` deliberately.** Too short (1s) burns CPU and inflates TSDB; too long (60s) makes percentile graphs jagged. The reference submission uses `5s` for a low-traffic lab; `15s` is the production default. Document your choice in §3.
  * **Выбирай `scrape_interval` осознанно.** Слишком короткий (1s) жжёт CPU и раздувает TSDB; слишком длинный (60s) делает графики перцентилей рваными. В эталонной сдаче для низконагруженной лабы используется `5s`; `15s` — продакшен-значение по умолчанию. Задокументируй выбор в §3.

<details>
<summary>💡 Optional second target — the course `echo` plumbing</summary>

💡 Необязательная вторая цель — сервис `echo` из репозитория курса

The repo ships a tiny Go service at `plumbing/echo` (you'll meet it again in Lab 9) that already exposes its own metrics on `/metrics`. Adding it gives your dashboard a second instrumented service — useful when you build the `sum by (job)` panels:

В репозитории есть маленький сервис на Go в `plumbing/echo` (встретишься с ним снова в лабе 9), который уже отдаёт собственные метрики на `/metrics`. Если добавить его, у дашборда появится второй инструментированный сервис — пригодится для панелей `sum by (job)`:

Add an `echo` service (`build: ../plumbing/echo`, port `8081`, on the `logging` network) and a fifth scrape job pointing at `echo:8081`. Same pattern, no new concepts.

Добавь сервис `echo` (`build: ../plumbing/echo`, порт `8081`, в сети `logging`) и пятый job сбора, нацеленный на `echo:8081`. Тот же шаблон, никаких новых концепций.

Not required for full marks; nice for the dashboard polish in Task 3.

Для полного балла не требуется; хорошо для полировки дашборда в задании 3.

</details>

### 2.4 — Bring it up and verify

2.4 — Поднять и проверить

```bash
cd monitoring
docker compose up -d
docker compose ps
```

Пояснение: переходим в каталог `monitoring`, поднимаем весь стек в фоне и смотрим статус контейнеров.

Verify the proof-of-work commands you'll paste in §2.5:

Проверь команды для доказательств, которые вставишь в §2.5:

```bash
# Prometheus is up
curl -s localhost:9090/-/healthy           # expect: "Prometheus Server is Healthy."

# Every job's target is UP — this is the proof your scrape config + service names are right.
# Run this AFTER one scrape_interval has elapsed.
curl -s localhost:9090/api/v1/targets \
  | jq -r '.data.activeTargets[] | "\(.labels.job): \(.health)"'
# (illustrative — every line must say "up"; if any says "down", that job's target field is wrong)
# <your_app_job>: up                       # YOUR TASK: this is the headline assertion —
# loki: up                                 #            the line for the job_name you picked in §2.3
# grafana: up                              #            for your app MUST say "up"
# prometheus: up
```

Пояснение: `/-/healthy` проверяет живость самого Prometheus; запрос к `/api/v1/targets` через `jq` печатает имя job и состояние каждой цели. Все строки должны быть `up`.

**Troubleshooting decision tree:**

**Дерево диагностики:**

- Target **down** → service name wrong, port wrong, or that service isn't running. `docker compose logs prometheus | grep -i scrape` shows the actual URL Prometheus tried.
  * Цель **down** → неверное имя сервиса, неверный порт или сервис не запущен. `docker compose logs prometheus | grep -i scrape` покажет реальный URL, который пробовал Prometheus.
- **No target at all** for a job → YAML indentation error in `prometheus.yml`. `docker compose logs prometheus` shows the parse error on startup.
  * **Цели вообще нет** для job → ошибка отступов YAML в `prometheus.yml`. `docker compose logs prometheus` покажет ошибку разбора при старте.
- Target **unknown** → wait one `scrape_interval` and refresh; the first scrape hasn't happened yet.
  * Цель **unknown** → подожди один `scrape_interval` и обнови; первый сбор ещё не произошёл.

### 2.5 — Proof of work

2.5 — Доказательства работы

**Paste into `docs/LAB08.md`:**

**Вставь в `docs/LAB08.md`:**

- `docker compose ps` showing `prometheus` `Up` and `(healthy)` (the latter after start_period elapses).
  * Вывод `docker compose ps`, где `prometheus` в состоянии `Up` и `(healthy)` (последнее — после истечения `start_period`).
- The exact `curl -s localhost:9090/api/v1/targets | jq ...` output from above — every job line must say `up`, with **`app: up`** clearly present. (This is the headline grading criterion for Task 2.)
  * Точный вывод `curl -s localhost:9090/api/v1/targets | jq ...` из примера выше — каждая строка job должна говорить `up`, причём **`app: up`** явно присутствует. (Это главный критерий оценки задания 2.)
- Screenshot of `http://localhost:9090/targets` confirming the same (UI version of the above).
  * Скриншот `http://localhost:9090/targets`, подтверждающий то же самое (версия в UI).
- Your three written answers from §2.1.
  * Твои три письменных ответа из §2.1.

---

## Task 3 — Grafana RED dashboard (2 pts)

Задание 3 — RED-дашборд Grafana (2 балла)

### 3.1 — Add Prometheus as a data source

3.1 — Добавить Prometheus как источник данных

In the Grafana from Lab 7:

В Grafana из лабы 7:

1. **Connections → Data sources → Add data source → Prometheus**.
   * **Connections → Data sources → Add data source → Prometheus** — добавь источник данных Prometheus.
2. URL: `http://prometheus:9090` — the service name on the `logging` network.
   * URL: `http://prometheus:9090` — имя сервиса в сети `logging`.
3. **Save & Test** → should report green.
   * **Save & Test** → статус должен стать зелёным.

> The bonus track provisions this automatically; doing it by hand once teaches you what the provisioning YAML encodes.
>
> Дополнительный трек настраивает это автоматически; если один раз сделать вручную, становится понятно, что именно кодирует YAML провижининга.

### 3.2 — Write the PromQL (`YOUR TASK`)

3.2 — Написать PromQL (`YOUR TASK`)

This is the heart of the lab. The lecture (slides 16, 18, 21) covers every function you need; you write the queries yourself.

Это сердце лабы. Лекция (слайды 16, 18, 21) разбирает все нужные функции; запросы пишешь ты сам.

For each row, write a PromQL expression that answers the question. Verify by running it in **Explore** (Prometheus data source) against real traffic — generate some first:

Для каждой строки напиши выражение PromQL, отвечающее на вопрос. Проверяй его в **Explore** (источник данных Prometheus) на реальном трафике — сначала сгенерируй его:

```bash
for i in $(seq 1 200); do curl -s localhost:8080/ > /dev/null; curl -s localhost:8080/health > /dev/null; done
```

Пояснение: генерируем трафик на оба эндпоинта, чтобы запросам было на чём проверяться.

| Question (RED) | Hint — one named function | Constraint |
|---|---|---|
| **Q1 — Rate.** Requests per second, broken down per endpoint. | `rate` | `sum by (endpoint) (...)` to keep `endpoint`, drop the rest |
| **Q2 — Errors.** 5xx requests per second, per endpoint. | `rate` with a `status=~"5.."` matcher | Same shape as Q1; the matcher goes **inside** the selector before `rate` |
| **Q3 — Duration.** p95 latency per endpoint. | `histogram_quantile` | The `sum by (endpoint, le)` goes **inside** `histogram_quantile`; never the reverse |
| **Q4 — Error ratio.** 5xx as a *fraction* of total requests. | Two `rate`s, divided | No grouping — single number; multiply by 100 for a percentage if you like |

| Вопрос (RED) | Подсказка — одна именованная функция | Ограничение |
|---|---|---|
| **Q1 — Rate.** Запросы в секунду с разбивкой по эндпоинтам. | `rate` | `sum by (endpoint) (...)` сохраняет `endpoint`, остальное отбрасывает |
| **Q2 — Errors.** 5xx-запросы в секунду по эндпоинтам. | `rate` с матчером `status=~"5.."` | Та же форма, что у Q1; матчер стоит **внутри** селектора до `rate` |
| **Q3 — Duration.** p95-латентность по эндпоинтам. | `histogram_quantile` | `sum by (endpoint, le)` идёт **внутри** `histogram_quantile`, никогда наоборот |
| **Q4 — Error ratio.** 5xx как *доля* от всех запросов. | Два `rate`, поделённые друг на друга | Без группировки — одно число; умножь на 100, если нужен процент |

`YOUR TASK`: write all four queries. Don't copy from the lecture verbatim — match them to *your* metric names from §1.3.

`YOUR TASK`: напиши все четыре запроса. Не копируй из лекции дословно — подгони их под *свои* имена метрик из §1.3.

```promql
# Q1 — Rate (requests/sec per endpoint)
___                              # YOUR TASK: rate(...) + sum by (endpoint)

# Q2 — Errors (5xx requests/sec per endpoint)
___                              # YOUR TASK: same shape, status=~"5.." selector

# Q3 — Duration (p95 latency per endpoint)
___                              # YOUR TASK: histogram_quantile OUTSIDE sum by (endpoint, le)

# Q4 — Error ratio (5xx fraction of total)
___                              # YOUR TASK: rate(5xx) / rate(total) — no grouping
```

Пояснение: заполни `___` четырьмя запросами — `rate` с `sum by (endpoint)`, вариант с матчером `status=~"5.."`, `histogram_quantile` поверх `sum by (endpoint, le)` и отношение двух `rate` без группировки.

Notes (don't skip):

Замечания (не пропускай):

- **`rate()`** handles counter resets (process restarts); `irate()` and raw subtraction do **not**. Stick with `rate()` for graphs.
  * **`rate()`** корректно обрабатывает сбросы счётчиков (перезапуски процесса); `irate()` и вычитание сырых значений — **нет**. Для графиков используй `rate()`.
- **Range floor:** with `scrape_interval: 5s`, `rate(...[1m])` is the floor that doesn't lie. With `15s`, use `[5m]` or longer.
  * **Нижняя граница диапазона:** при `scrape_interval: 5s` минимальный честный диапазон — `rate(...[1m])`. При `15s` используй `[5m]` или больше.
- **Never average a percentile.** `histogram_quantile()` goes *outside* the `sum by (le, …)`, never inside an `avg()`. Try the wrong nesting once and read Prometheus's error — it's instructive.
  * **Никогда не усредняй перцентиль.** `histogram_quantile()` стоит *снаружи* `sum by (le, …)`, а не внутри `avg()`. Попробуй один раз неправильную вложенность и прочитай ошибку Prometheus — это поучительно.
- **Use the lecture's metric name shape** if you're stuck on naming (`http_requests_total` etc.) — but Q1–Q4 must match the metric names you actually exposed in §1.3.
  * **Используй форму имён метрик из лекции**, если застрял на именовании (`http_requests_total` и т. п.), но Q1–Q4 должны соответствовать именам метрик, которые ты реально выставил в §1.3.

### 3.3 — Build the dashboard — 6 panels

3.3 — Построить дашборд — 6 панелей

Each panel answers one question. PromQL queries 1, 2, 3, 5, 6 are described below; panel 4 (heatmap) is one you write yourself in §3.2-extended.

Каждая панель отвечает на один вопрос. Запросы PromQL 1, 2, 3, 5, 6 описаны ниже; панель 4 (heatmap) ты напишешь сам в расширении §3.2.

| # | Panel | Visualisation | Query source | Answers |
|---|-------|---------------|---|---|
| 1 | **Request rate** | Time series | Your Q1 | *How busy?* |
| 2 | **Error rate** | Time series | Your Q2 | *How often failing?* |
| 3 | **p95 latency** | Time series | Your Q3 | *How slow at the 95th?* |
| 4 | **Latency heatmap** | Heatmap | YOUR TASK — same `_bucket` series, but `sum by (le)` *without* `histogram_quantile` (the heatmap panel computes percentiles itself) | *What does the long tail look like?* |
| 5 | **In-flight or total** | Gauge / Stat | `sum(rate(<your counter>[1m]))` | *How busy right now?* |
| 6 | **Service uptime** | Stat | `up{job="<your app job name>"}` | *Are we alive?* |

| # | Панель | Визуализация | Источник запроса | Отвечает на вопрос |
|---|-------|-------|---|---|
| 1 | **Частота запросов (Request rate)** | Временной ряд | Твой Q1 | *Насколько загружен?* |
| 2 | **Частота ошибок (Error rate)** | Временной ряд | Твой Q2 | *Как часто падает?* |
| 3 | **Латентность p95 (p95 latency)** | Временной ряд | Твой Q3 | *Насколько медленно на 95-м перцентиле?* |
| 4 | **Тепловая карта латентности (Latency heatmap)** | Тепловая карта | `YOUR TASK` — те же серии `_bucket`, но `sum by (le)` *без* `histogram_quantile` (панель heatmap вычисляет перцентили сама) | *Как выглядит длинный хвост?* |
| 5 | **Запросы в обработке или всего (In-flight or total)** | Gauge / Stat | `sum(rate(<твой счётчик>[1m]))` | *Насколько загружен прямо сейчас?* |
| 6 | **Доступность сервиса (Service uptime)** | Stat | `up{job="<имя job твоего приложения>"}` | *Мы живы?* |

`YOUR TASK`: write panel 4's PromQL yourself — it's the only one not covered by Q1–Q4. The hint: the heatmap **does not** call `histogram_quantile`; it visualises the bucket counts directly. Same `sum by (le) (rate(...))` shape, no quantile wrapper.

`YOUR TASK`: напиши PromQL для панели 4 сам — это единственная панель, не покрытая Q1–Q4. Подсказка: heatmap **не** вызывает `histogram_quantile`; она визуализирует счётчики бакетов напрямую. Та же форма `sum by (le) (rate(...))`, без обёртки квантиля.

**How to build (Grafana 13):**

**Как строить (Grafana 13):**

1. **Dashboards → New → New dashboard → Add visualization** → pick the **Prometheus** data source.
   * **Dashboards → New → New dashboard → Add visualization** → выбери источник данных **Prometheus**.
2. Paste the PromQL; set the visualisation type from the table; title each panel **after the question it answers** ("Errors per second by endpoint" beats "Query 2").
   * Вставь PromQL; выставь тип визуализации из таблицы; назови каждую панель **по вопросу, на который она отвечает** («Errors per second by endpoint» лучше, чем «Query 2»).
3. Set sensible units: panel 1 = `req/s`; panel 3 = `seconds`; panels 2 = `req/s` with a threshold so non-zero errors turn the panel red.
   * Задай осмысленные единицы: панель 1 = `req/s`; панель 3 = `seconds`; панель 2 = `req/s` с порогом, чтобы ненулевые ошибки красили панель в красный.
4. Use `{{endpoint}}` (or `{{job}}`) in the legend so the lines are labelled, not "Series 1".
   * Используй `{{endpoint}}` (или `{{job}}`) в легенде, чтобы линии были подписаны, а не «Series 1».
5. **Save dashboard**, then **Share → Export → Save to file**, commit to `monitoring/grafana/dashboards/lab08.json`.
   * **Save dashboard**, затем **Share → Export → Save to file**, закоммить в `monitoring/grafana/dashboards/lab08.json`.

> 💡 Want a head start on infrastructure panels? Import community dashboard **ID 3662** (Prometheus self-stats) via **Dashboards → New → Import** and bind it to your Prometheus data source. Your *own* 6-panel RED dashboard is what's graded.
>
> 💡 Хочешь готовый старт для инфраструктурных панелей? Импортируй community-дашборд **ID 3662** (самостатистика Prometheus) через **Dashboards → New → Import** и привяжи к своему источнику данных Prometheus. Оценивается именно твой *собственный* 6-панельный RED-дашборд.

### 3.4 — Proof of work

3.4 — Доказательства работы

**Paste into `docs/LAB08.md`:**

**Вставь в `docs/LAB08.md`:**

- Screenshot of the 6-panel dashboard with **live data on every panel** (generate fresh traffic right before the screenshot — empty panels = no credit).
  * Скриншот 6-панельного дашборда с **живыми данными на каждой панели** (сгенерируй свежий трафик прямо перед скриншотом — пустые панели = 0 баллов).
- The PromQL queries Q1–Q4 + the panel-4 heatmap query in a markdown table, each labelled with the question it answers.
  * Запросы PromQL Q1–Q4 + запрос heatmap для панели 4 в markdown-таблице, каждый с подписью вопроса, на который он отвечает.
- A real PromQL response from your own traffic, captured via the HTTP API (so the grader sees *your* numbers, not just a screenshot). Run the four queries through `/api/v1/query` and capture the output:
  * Реальный ответ PromQL на твоём трафике, снятый через HTTP API (чтобы проверяющий видел *твои* числа, а не только скриншот). Прогони четыре запроса через `/api/v1/query` и сохрани вывод:
  ```bash
  curl -s --get http://localhost:9090/api/v1/query \
       --data-urlencode 'query=___' \
       | jq '.data.result'
  # YOUR TASK: substitute each of your Q1–Q4 queries in turn; paste the .data.result
  # (illustrative — your counts will differ)
  # [{"metric":{"endpoint":"/"},"value":[1716894123.456,"127"]},
  #  {"metric":{"endpoint":"/health"},"value":[1716894123.456,"54"]}]
  ```
- The committed JSON file path: `monitoring/grafana/dashboards/lab08.json`.
  * Путь к закоммиченному JSON-файлу: `monitoring/grafana/dashboards/lab08.json`.

---

## Task 4 — Production readiness (1 pt)

Задание 4 — Готовность к продакшену (1 балл)

Harden the stack so it isn't a toy. Everything here builds on the `prometheus` service you wrote in §2.2 — the blanks are already there, this task is where you *justify* the values you picked.

Укрепи стек, чтобы он не был игрушечным. Всё здесь опирается на сервис `prometheus` из §2.2 — пустые места уже есть, в этом задании ты *обосновываешь* выбранные значения.

### 4.1 — Resource limits

4.1 — Лимиты ресурсов

`YOUR TASK`: add `deploy.resources` to the `prometheus` service. Pick sane numbers — Prometheus's RAM scales with active series count; for this lab a couple of hundred MiB is generous. Document the numbers + your reasoning in `LAB08.md`.

`YOUR TASK`: добавь `deploy.resources` в сервис `prometheus`. Выбери разумные числа — ОЗУ Prometheus растёт с числом активных серий; для этой лабы нескольких сотен MiB более чем достаточно. Задокументируй числа и обоснование в `LAB08.md`.

```yaml
    deploy:
      resources:
        limits:
          cpus: ___                 # YOUR TASK: ceiling — what's the most you'd let it eat?
          memory: ___               # YOUR TASK: RAM ceiling — keep series count in mind
        reservations:
          cpus: ___                 # YOUR TASK: guaranteed floor at scheduling time
          memory: ___               # YOUR TASK: guaranteed memory floor
```

Пояснение: `limits` задают потолок CPU и памяти, `reservations` — гарантированный минимум при планировании.

### 4.2 — Retention

4.2 — Срок хранения данных (retention)

You stubbed the retention flags in §2.2. Document **why both** `retention.time` AND `retention.size` matter in `LAB08.md`: time is the SLA-driven knob, size is the defence-in-depth one against runaway cardinality.

Ты заготовил флаги retention в §2.2. Задокументируй в `LAB08.md`, **почему важны оба** — `retention.time` И `retention.size`: время — параметр, привязанный к SLA, размер — защита в глубину от неконтролируемой кардинальности.

### 4.3 — Health check & persistence

4.3 — Healthcheck и персистентность

The healthcheck block from §2.2 should be filled in. Verify it works:

Блок healthcheck из §2.2 должен быть заполнен. Проверь, что он работает:

```bash
docker compose ps                              # prometheus reports (healthy) once start_period elapses
docker compose down && docker compose up -d    # NOTE: no `-v` — that would wipe the named volumes
sleep 30                                       # wait for the first post-restart scrape (≥ one scrape_interval + start_period)
curl -s 'http://localhost:9090/api/v1/query?query=app_requests_total' | jq '.data.result | length'
# (illustrative — non-zero means your TSDB volume persisted the series through the restart;
#  exactly 0 means either the volume wasn't mounted or `down -v` was used by mistake)
```

Пояснение: `down` без `-v` не удаляет именованные тома; после повторного `up` и ожидания запрос должен вернуть ненулевое число серий — значит, история метрик сохранилась.

### 4.4 — Proof of work

4.4 — Доказательства работы

**Paste into `docs/LAB08.md`:**

**Вставь в `docs/LAB08.md`:**

- `docker compose ps` output showing **all** services healthy, including `prometheus (healthy)`.
  * Вывод `docker compose ps`, где **все** сервисы здоровы, включая `prometheus (healthy)`.
- Output of the `down` → `up` → query sequence above proving metric history survived.
  * Вывод последовательности `down` → `up` → запрос выше, доказывающий, что история метрик выжила.
- Your resource-limit values + one-sentence reasoning for each.
  * Значения лимитов ресурсов + обоснование каждого одним предложением.

---

## Task 5 — Documentation (1 pt)

Задание 5 — Документация (1 балл)

`YOUR TASK`: write `monitoring/docs/LAB08.md` with these sections, in order:

`YOUR TASK`: напиши `monitoring/docs/LAB08.md` с этими разделами по порядку:

1. **Architecture** — a Mermaid diagram of `app → Prometheus → Grafana`, alongside the Lab 7 log path (so the same Grafana shows both pillars).
   * **Архитектура** — Mermaid-схема `app → Prometheus → Grafana` рядом с путём логов из лабы 7 (чтобы одна и та же Grafana показывала оба столпа).
2. **Setup** — how to deploy (`docker compose up -d`) and the verification commands from §2.4.
   * **Настройка** — как развернуть (`docker compose up -d`) и команды проверки из §2.4.
3. **Instrumentation** — the metric names + types + labels you added, plus your answers from §1.1.
   * **Инструментирование** — имена, типы и метки добавленных метрик, плюс твои ответы из §1.1.
4. **Scrape config** — your job names, your `scrape_interval`, the pull model (answer the §2.1 questions).
   * **Конфиг сбора** — твои имена job, твой `scrape_interval`, модель pull (ответы на вопросы §2.1).
5. **Dashboard** — the table from §3.4: question → PromQL → screenshot.
   * **Дашборд** — таблица из §3.4: вопрос → PromQL → скриншот.
6. **Production config** — retention rationale, resource limits, healthcheck, persistence proof.
   * **Продакшен-конфиг** — обоснование retention, лимиты ресурсов, healthcheck, доказательство персистентности.
7. **Metrics vs logs** — one paragraph: when you'd reach for Prometheus vs the Lab 7 LogQL query.
   * **Метрики против логов** — один абзац: когда обращаться к Prometheus, а когда к LogQL-запросу из лабы 7.
8. **Challenges & solutions** — at least one real one (not "I was new to Prometheus").
   * **Трудности и решения** — хотя бы одна настоящая (не «я был новичком в Prometheus»).

Include config snippets (not whole files) and the captures from Tasks 1–4. Keep it readable — this is the artefact your future on-call self will read at 3 am.

Включай фрагменты конфигов (не целые файлы) и доказательства из заданий 1–4. Пиши читаемо — этот артефакт твой будущий дежурный «я» прочитает в 3 часа ночи.

---

## Bonus Task — Ansible automation (2 pts)

Бонусное задание — автоматизация Ansible (2 балла)

Extend the `roles/monitoring` role from Lab 6/7 so a single playbook deploys **logs + metrics** together.

Расширь роль `roles/monitoring` из лаб 6/7, чтобы один плейбук разворачивал **логи + метрики** вместе.

`YOUR TASK`: update the role to:

`YOUR TASK`: обнови роль так, чтобы она:

- **Template** `prometheus/prometheus.yml` from Jinja2 — scrape interval, retention, **and the target list** become role variables (defined in `defaults/main.yml`). The same template renders correctly whether you have 4 targets or 14.
  * **Шаблонизировала** `prometheus/prometheus.yml` через Jinja2 — интервал сбора, retention **и список целей** становятся переменными роли (определены в `defaults/main.yml`). Один и тот же шаблон должен корректно рендериться и для 4 целей, и для 14.
- **Extend** the templated `docker-compose.yml` from Lab 7 with the `prometheus` service + `prometheus-data` volume.
  * **Расширила** шаблонный `docker-compose.yml` из лабы 7 сервисом `prometheus` и томом `prometheus-data`.
- **Provision** the Prometheus data source into Grafana via `grafana/provisioning/datasources/`.
  * **Провижинила** источник данных Prometheus в Grafana через `grafana/provisioning/datasources/`.
- **Provision** your exported RED dashboard JSON via `grafana/provisioning/dashboards/`.
  * **Провижинила** экспортированный JSON RED-дашборда через `grafana/provisioning/dashboards/`.
- **Wait** for Prometheus `:9090/-/healthy` before reporting success — same pattern as the Lab 7 readiness wait, just a different URL.
  * **Дожидалась** `:9090/-/healthy` Prometheus перед сообщением об успехе — тот же шаблон ожидания готовности, что в лабе 7, только другой URL.
- Be **idempotent**: a second run reports `changed=0`.
  * Была **идемпотентной**: повторный запуск сообщает `changed=0`.

Less hand-holding than Tasks 1–5: figure out the variable names, the template structure, and the readiness polling yourself. Lab 7's bonus already covered the `docker_compose_v2` deploy mechanics + the readiness loop pattern.

Подсказок меньше, чем в заданиях 1–5: имена переменных, структуру шаблона и опрос готовности придумай сам. Бонус лабы 7 уже покрывал механику деплоя через `docker_compose_v2` и шаблон цикла ожидания готовности.

**Variables you'll need at minimum** (don't quote-and-paste — declare in your `defaults/main.yml`):

**Переменные, которые понадобятся как минимум** (не копируй дословно — объяви в своём `defaults/main.yml`):

- `prometheus_version` (e.g. `v3.11.3`), `prometheus_port` (`9090`)
  * `prometheus_version` (например, `v3.11.3`), `prometheus_port` (`9090`)
- `prometheus_retention_time` (`15d`), `prometheus_retention_size` (`2GB`)
  * `prometheus_retention_time` (`15d`), `prometheus_retention_size` (`2GB`)
- `prometheus_scrape_interval` (`15s`)
  * `prometheus_scrape_interval` (`15s`)
- `prometheus_targets` — a list of `{job, targets}` dicts, so a Jinja2 `{% for %}` renders the scrape config
  * `prometheus_targets` — список словарей `{job, targets}`, чтобы Jinja2-цикл `{% for %}` отрендерил конфиг сбора

**Evidence (paste into `docs/LAB08.md`):**

**Доказательства (вставь в `docs/LAB08.md`):**

- First-run output (changes > 0) and second-run output (`changed=0`).
  * Вывод первого запуска (changes > 0) и второго (`changed=0`).
- The **rendered** (not template) `prometheus.yml` from a real run.
  * **Отрендеренный** (не шаблонный) `prometheus.yml` из реального запуска.
- Screenshot of Grafana showing **both** data sources (Loki + Prometheus) working with traffic.
  * Скриншот Grafana, где **оба** источника данных (Loki + Prometheus) работают с трафиком.
- Path: `ansible/roles/monitoring/` + `ansible/playbooks/deploy-monitoring.yml`.
  * Путь: `ansible/roles/monitoring/` + `ansible/playbooks/deploy-monitoring.yml`.

---

## How to Submit

Как сдавать

```bash
git switch -c lab08
git add app_python/                                # instrumentation + requirements
git add monitoring/                                # compose + prometheus.yml + dashboard
git add ansible/roles/monitoring \
        ansible/playbooks/deploy-monitoring.yml    # only if bonus done
git commit -m "feat(lab08): prometheus metrics + RED dashboard"
git push -u origin lab08
```

Пояснение: создаём ветку `lab08`, добавляем приложение, стек мониторинга и (при выполнении бонуса) файлы Ansible, коммитим и пушим ветку.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab08` → `course-repo:master` *(reviewed)*
  * `your-fork:lab08` → `course-repo:master` *(с ревью)*
- `your-fork:lab08` → `your-fork:master` *(merges into your own main)*
  * `your-fork:lab08` → `your-fork:master` *(вливается в твой main)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — Counter + Histogram + /metrics + before/after hooks; /metrics excluded from self-counting
- [ ] Task 2 done — prometheus service in compose, prometheus.yml filled, all targets `up`
- [ ] Task 3 done — 6-panel RED dashboard, JSON committed, real PromQL response captured
- [ ] Task 4 done — limits, retention, healthcheck healthy, persistence proven
- [ ] Task 5 done — LAB08.md with all 8 sections + captures
- [ ] Bonus done — idempotent Ansible role with readiness wait, both data sources working
```

Пояснение: чек-лист сдачи — отметь пункты по заданиям 1–5 и бонусу; каждый пункт соответствует критерию приёмки.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Instrumentation (3 pts)

Задание 1 — Инструментирование (3 балла)

- ✅ `app_requests_total` Counter with labels `method, endpoint, status` declared
  * ✅ Counter `app_requests_total` с метками `method, endpoint, status` объявлен
- ✅ `app_request_duration_seconds` Histogram with labels `method, endpoint` declared, SRE-default buckets
  * ✅ Histogram `app_request_duration_seconds` с метками `method, endpoint` объявлен, бакеты SRE по умолчанию
- ✅ `before_request` / `after_request` hooks observe **every** non-`/metrics` request
  * ✅ Хуки `before_request` / `after_request` наблюдают **каждый** запрос, кроме `/metrics`
- ✅ `/metrics` returns valid Prometheus text format with the right `Content-Type`
  * ✅ `/metrics` возвращает корректный текстовый формат Prometheus с правильным `Content-Type`
- ✅ `/metrics` does **not** count itself (`grep -c 'endpoint="/metrics"' = 0`)
  * ✅ `/metrics` **не** считает сам себя (`grep -c 'endpoint="/metrics"' = 0`)
- ✅ Three written answers from §1.1 in `LAB08.md`
  * ✅ Три письменных ответа из §1.1 в `LAB08.md`

### Task 2 — Prometheus scrape (3 pts)

Задание 2 — Сбор метрик Prometheus (3 балла)

- ✅ `prometheus` service added on the `logging` network with retention flags + healthcheck
  * ✅ Сервис `prometheus` добавлен в сеть `logging` с флагами retention и healthcheck
- ✅ `prometheus.yml` has `global` + 4 scrape jobs filled from the scaffold
  * ✅ В `prometheus.yml` есть `global` + 4 job-а сбора, заполненные из каркаса
- ✅ `curl -s localhost:9090/api/v1/targets | jq` shows **every** job `up`, with `app: up` clearly present
  * ✅ `curl -s localhost:9090/api/v1/targets | jq` показывает **каждый** job в состоянии `up`, причём `app: up` явно присутствует
- ✅ Three written answers from §2.1 in `LAB08.md`
  * ✅ Три письменных ответа из §2.1 в `LAB08.md`

### Task 3 — RED dashboard (2 pts)

Задание 3 — RED-дашборд (2 балла)

- ✅ 6 panels with appropriate visualisations; titles describe the question, not the query
  * ✅ 6 панелей с подходящими визуализациями; заголовки описывают вопрос, а не запрос
- ✅ Q1–Q4 PromQL + panel-4 heatmap query all written by the student (not copy-pasted from the lecture)
  * ✅ PromQL Q1–Q4 + запрос heatmap для панели 4 написаны студентом (не скопированы из лекции)
- ✅ A real PromQL response from the student's own traffic captured via `/api/v1/query`
  * ✅ Реальный ответ PromQL на трафике студента, снятый через `/api/v1/query`
- ✅ Dashboard JSON committed to `monitoring/grafana/dashboards/lab08.json`
  * ✅ JSON дашборда закоммичен в `monitoring/grafana/dashboards/lab08.json`

### Task 4 — Production config (1 pt)

Задание 4 — Продакшен-конфиг (1 балл)

- ✅ Resource limits + reservations on `prometheus`
  * ✅ Лимиты и резервации ресурсов у `prometheus`
- ✅ Retention flags (`time` and `size`) set on the command line
  * ✅ Флаги retention (`time` и `size`) заданы в командной строке
- ✅ Healthcheck reports `(healthy)`; metric history survives `down`/`up`
  * ✅ Healthcheck сообщает `(healthy)`; история метрик выживает `down`/`up`

### Task 5 — Docs (1 pt)

Задание 5 — Документация (1 балл)

- ✅ All 8 sections present in `monitoring/docs/LAB08.md`
  * ✅ Все 8 разделов присутствуют в `monitoring/docs/LAB08.md`
- ✅ Research answers in the student's own words, not lecture quotes
  * ✅ Ответы на исследовательские вопросы своими словами, а не цитаты из лекции

### Bonus — Ansible (2 pts)

Бонус — Ansible (2 балла)

- ✅ `prometheus.yml` templated from a `prometheus_targets` list variable
  * ✅ `prometheus.yml` шаблонизирован из переменной-списка `prometheus_targets`
- ✅ Data source + dashboard provisioned (no clicks in the Grafana UI)
  * ✅ Источник данных и дашборд провижинятся (без кликов в UI Grafana)
- ✅ Deploy is idempotent — second run `changed=0`
  * ✅ Деплой идемпотентен — второй запуск `changed=0`
- ✅ Readiness wait blocks success until Prometheus + Grafana respond
  * ✅ Ожидание готовности не даёт сообщить об успехе, пока Prometheus и Grafana не ответят
- ✅ Both data sources visible in Grafana
  * ✅ Оба источника данных видны в Grafana

---

## Rubric

Рубрика

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Instrumentation | **3** | Counter + Histogram declared correctly; hooks observe non-`/metrics` requests; valid exposition format |
| **Task 2** — Prometheus scrape | **3** | Service in compose, scrape config filled, `/api/v1/targets` shows `app: up` |
| **Task 3** — RED dashboard | **2** | 6 panels with student-written PromQL; real query response captured; JSON committed |
| **Task 4** — Production config | **1** | Limits, retention, healthcheck, persistence |
| **Task 5** — Docs | **1** | All 8 sections, captures present, answers in student's own words |
| **Bonus** — Ansible | **2** | Idempotent templated deployment of the full logs+metrics stack |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Инструментирование | **3** | Counter + Histogram объявлены правильно; хуки учитывают все запросы, кроме `/metrics`; корректный формат экспозиции |
| **Задание 2** — Сбор метрик Prometheus | **3** | Сервис в compose, конфиг сбора заполнен, `/api/v1/targets` показывает `app: up` |
| **Задание 3** — RED-дашборд | **2** | 6 панелей с PromQL, написанным студентом; захвачен реальный ответ запроса; JSON закоммичен |
| **Задание 4** — Продакшен-конфиг | **1** | Лимиты, retention, healthcheck, персистентность |
| **Задание 5** — Документация | **1** | Все 8 разделов, есть скриншоты, ответы своими словами |
| **Бонус** — Ansible | **2** | Идемпотентный шаблонный деплой полного стека логов и метрик |
| **Итого** | **12** | 10 основных + 2 бонусных |

---

## Resources

Ресурсы

<details>
<summary>📊 Prometheus</summary>

📊 Prometheus — официальная документация

- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
  * [Обзор Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Prometheus 3.0 announcement](https://prometheus.io/blog/2024/11/14/prometheus-3-0/) — UTF-8, native histograms, OTLP
  * [Анонс Prometheus 3.0](https://prometheus.io/blog/2024/11/14/prometheus-3-0/) — UTF-8, нативные гистограммы, OTLP
- [Configuration reference](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
  * [Справочник по конфигурации](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [`scrape_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
  * [`scrape_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)
- [Metric types](https://prometheus.io/docs/concepts/metric_types/)
  * [Типы метрик](https://prometheus.io/docs/concepts/metric_types/)
- [Native histograms](https://prometheus.io/docs/specs/native_histograms/)
  * [Нативные гистограммы](https://prometheus.io/docs/specs/native_histograms/)

</details>

<details>
<summary>🐍 Python instrumentation</summary>

🐍 Инструментирование Python

- [`prometheus_client` (GitHub)](https://github.com/prometheus/client_python)
  * [`prometheus_client` (GitHub)](https://github.com/prometheus/client_python)
- [Python instrumentation guide](https://prometheus.io/docs/guides/python/)
  * [Руководство по инструментированию Python](https://prometheus.io/docs/guides/python/)
- [Instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
  * [Лучшие практики инструментирования](https://prometheus.io/docs/practices/instrumentation/)
- [Metric naming](https://prometheus.io/docs/practices/naming/)
  * [Именование метрик](https://prometheus.io/docs/practices/naming/)

</details>

<details>
<summary>📈 PromQL & Grafana</summary>

📈 PromQL и Grafana

- [PromQL basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
  * [Основы PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [PromQL examples](https://prometheus.io/docs/prometheus/latest/querying/examples/)
  * [Примеры PromQL](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- [Grafana Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
  * [Источник данных Prometheus в Grafana](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [Dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)
  * [Провижининг дашбордов](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)

</details>

<details>
<summary>📚 Observability methods</summary>

📚 Методы наблюдаемости

- [RED method (Tom Wilkie, Grafana)](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)
  * [Метод RED (Tom Wilkie, Grafana)](https://grafana.com/blog/the-red-method-how-to-instrument-your-services/)
- [USE method (Brendan Gregg)](https://www.brendangregg.com/usemethod.html)
  * [Метод USE (Brendan Gregg)](https://www.brendangregg.com/usemethod.html)
- [The Four Golden Signals (Google SRE)](https://sre.google/sre-book/monitoring-distributed-systems/)
  * [Четыре золотых сигнала (Google SRE)](https://sre.google/sre-book/monitoring-distributed-systems/)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **The `/metrics` self-counting trap.** If your `after_request` hook records the response to `/metrics` itself, every scrape increments `app_requests_total{endpoint="/metrics",...}` — and on a 5-second scrape interval that's twelve fake "requests per minute" that drown out your real traffic. **Always** exclude the `/metrics` path before incrementing; verify with `curl … | grep -c 'endpoint="/metrics"'` returning `0`. This is the #1 instrumentation bug — the ref submission's grading notes explicitly check for it.
  * **Ловушка самосчёта `/metrics`.** Если хук `after_request` записывает и ответ на `/metrics`, каждый сбор увеличивает `app_requests_total{endpoint="/metrics",...}` — а при интервале сбора 5 секунд это двенадцать фальшивых «запросов в минуту», которые заглушают реальный трафик. **Всегда** исключай путь `/metrics` до инкремента; проверяй через `curl … | grep -c 'endpoint="/metrics"'`, который должен вернуть `0`. Это баг инструментирования №1 — в заметках к эталонной сдаче его проверяют явно.
- **Label cardinality = Prometheus OOM.** Each unique label combination is a time series (~3–5 KB RAM each). Putting `user_id`, `request_id`, `trace_id`, raw paths with IDs (`/user/123`), emails, or timestamps in a label is the fastest way to crash the ingester. **Use `request.url_rule.rule`** for the `endpoint` label — `request.path` includes `/user/123` and `/user/456` as two distinct series; the rule `/user/<id>` collapses them. Same lesson as Loki labels in Lab 7.
  * **Кардинальность меток = OOM Prometheus.** Каждая уникальная комбинация меток — временная серия (примерно 3–5 КБ ОЗУ на каждую). Положить в метку `user_id`, `request_id`, `trace_id`, сырые пути с ID (`/user/123`), email или временные метки — самый быстрый способ уронить ingester. **Используй `request.url_rule.rule`** для метки `endpoint`: `request.path` даст `/user/123` и `/user/456` как две разные серии, а правило `/user/<id>` их схлопнет. Тот же урок, что с метками Loki в лабе 7.
- **Counter vs Gauge confusion.** A counter only goes up (and resets to 0 on process restart). If you find yourself wanting `.dec()` or `.set()`, you reached for a counter when you needed a gauge. Conversely: querying a counter's raw value gives you "lifetime requests since boot" — almost never what you want. Always wrap with `rate()` or `increase()`.
  * **Путаница Counter и Gauge.** Счётчик только растёт (и сбрасывается в 0 при перезапуске процесса). Если тебе хочется `.dec()` или `.set()`, ты взял Counter там, где нужен Gauge. И наоборот: запрос сырого значения счётчика даёт «запросы за всё время с момента запуска» — почти никогда не то, что нужно. Всегда оборачивай в `rate()` или `increase()`.
- **Histogram bucket choice.** The default `prometheus_client` histogram buckets max out at **10 seconds**. If you're measuring database queries that can take 30 s, every slow query falls into the `+Inf` bucket and your p99 is a lie. Bucket your histograms for the *expected* range with one bucket past the tolerance ceiling — start with the SRE defaults `(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10)` and tune from production data. Buckets too narrow on the low end is the symmetric trap: every fast request lands in the smallest bucket and you can't tell 1ms from 50µs.
  * **Выбор бакетов гистограммы.** Стандартные бакеты гистограммы `prometheus_client` заканчиваются на **10 секундах**. Если ты измеряешь запросы к БД, которые могут длиться 30 с, каждый медленный запрос попадает в бакет `+Inf`, и твой p99 — ложь. Настраивай бакеты под *ожидаемый* диапазон, добавив один бакет выше порога допустимого — начни с набора SRE по умолчанию `(.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10)` и подстрой по продакшен-данным. Слишком узкие бакеты в нижней части — симметричная ловушка: каждый быстрый запрос попадает в самый маленький бакет, и ты не отличишь 1 мс от 50 мкс.
- **Compose service-name DNS.** Prometheus scrapes targets by hostname over HTTP. Inside the `logging` Compose network, the hostname is the **service name** (`app`, `loki`, `grafana`) — **not** `localhost` and **not** the host IP. The only target legitimately on `localhost` is Prometheus's self-scrape (because that traffic stays inside the Prometheus container). Get this wrong and `up{job="app"}` returns 0 with no obvious error in the UI; check `docker compose logs prometheus | grep -i scrape`.
  * **DNS имён сервисов в Compose.** Prometheus собирает метрики с целей по hostname через HTTP. Внутри сети Compose `logging` hostname — это **имя сервиса** (`app`, `loki`, `grafana`), а **не** `localhost` и **не** IP хоста. Единственная цель, для которой законно использовать `localhost`, — самосбор Prometheus (этот трафик не покидает контейнер Prometheus). Ошибёшься — и `up{job="app"}` вернёт 0 без очевидной ошибки в UI; проверяй `docker compose logs prometheus | grep -i scrape`.
- **Retention is a CLI flag, not a config-file setting.** This catches everyone migrating from a 2.x article. `retention_period` in `prometheus.yml` is **ignored** — set `--storage.tsdb.retention.time=15d` and `--storage.tsdb.retention.size=2GB` on the `command:` list instead. Both can be present; whichever triggers first wins.
  * **Retention — флаг CLI, а не настройка конфиг-файла.** На этом спотыкаются все, кто переходит со статьи про 2.x. `retention_period` в `prometheus.yml` **игнорируется** — задавай `--storage.tsdb.retention.time=15d` и `--storage.tsdb.retention.size=2GB` в списке `command:`. Можно указать оба; срабатывает то, что наступит раньше.
- **`scrape_interval` mismatch with PromQL window.** With `scrape_interval: 15s`, `rate(...[15s])` is degenerate (one or zero samples) and the graph is empty. The minimum query range that gives meaningful output is `≥ 4× scrape_interval`. With 5s scrapes → `[1m]` is safe; with 15s scrapes → `[5m]` is the floor.
  * **Несогласованность `scrape_interval` и окна PromQL.** При `scrape_interval: 15s` выражение `rate(...[15s])` вырождено (один или ноль сэмплов), и график пуст. Минимальный диапазон запроса, дающий осмысленный результат, — `≥ 4× scrape_interval`. При сборе раз в 5 с безопасен `[1m]`; при 15 с минимальный — `[5m]`.
- **`histogram_quantile` outside, `sum by (le)` inside — never the reverse.** The killer percentile query is `histogram_quantile(0.95, sum by (endpoint, le) (rate(*_bucket[5m])))`. Wrapping `histogram_quantile` *inside* an aggregator (e.g. `avg(histogram_quantile(...))`) is mathematical nonsense — averaging quantiles doesn't give you a quantile. Try it once, read the result, internalise.
  * **`histogram_quantile` снаружи, `sum by (le)` внутри — никогда наоборот.** Канонический запрос перцентиля: `histogram_quantile(0.95, sum by (endpoint, le) (rate(*_bucket[5m])))`. Оборачивать `histogram_quantile` *внутрь* агрегатора (например, `avg(histogram_quantile(...))`) — математическая бессмыслица: среднее перцентилей не даёт перцентиль. Попробуй один раз, прочитай результат, запомни.
- **`prometheus-client` >=0.23 emits `# HELP`, `# TYPE`, and `# UNIT`** — and Prometheus parses all three. If you wrote your own ad-hoc `/metrics` text by hand and skipped the `# TYPE` line, Prometheus silently treats every series as an untyped gauge, and `rate()` on it returns NaN. Always use `generate_latest()`.
  * **`prometheus-client` >=0.23 выводит `# HELP`, `# TYPE` и `# UNIT`** — и Prometheus разбирает все три. Если ты написал свой текст `/metrics` вручную и пропустил строку `# TYPE`, Prometheus молча считает каждую серию нетипизированным gauge, и `rate()` на ней вернёт NaN. Всегда используй `generate_latest()`.

</details>

<details>
<summary>🛠️ Dev tools worth knowing</summary>

🛠️ Полезные инструменты разработчика

- [`promtool`](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) — ships in the Prometheus image; `promtool check config /etc/prometheus/prometheus.yml` validates before reload
  * [`promtool`](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) — входит в образ Prometheus; `promtool check config /etc/prometheus/prometheus.yml` проверяет конфиг до перезагрузки
- [`hey`](https://github.com/rakyll/hey) — generate steady traffic against your app to populate dashboards (`hey -n 5000 -c 10 http://localhost:8080/`)
  * [`hey`](https://github.com/rakyll/hey) — генерирует ровный трафик к приложению, чтобы наполнить дашборды (`hey -n 5000 -c 10 http://localhost:8080/`)
- [`jq`](https://jqlang.github.io/jq/) — for parsing `/api/v1/targets` and `/api/v1/query` responses on the CLI
  * [`jq`](https://jqlang.github.io/jq/) — разбор ответов `/api/v1/targets` и `/api/v1/query` в командной строке

</details>

---

## Looking Ahead

Взгляд вперёд

| Lab | What it adds to this service |
|---:|---|
| 9 | k3d Kubernetes — deploy your instrumented app + the `echo` plumbing service as Pods, scrape via a sidecar |
| 10 | Helm 4 chart — package the app + Prometheus values |
| 12 | ConfigMaps + PVCs — re-mount Prometheus config from a ConfigMap; persist TSDB on a PVC |
| 16 | `kube-prometheus-stack` Helm chart with `ServiceMonitor` CRDs auto-discovering your app — same PromQL, K8s-native plumbing |

| Лаба | Что добавляет этому сервису |
|---:|---|
| 9 | k3d Kubernetes — разворачиваем инструментированное приложение и сервис `echo` как Pod-ы, собираем метрики через сайдкар |
| 10 | Чарт Helm 4 — упаковываем приложение + values для Prometheus |
| 12 | ConfigMaps + PVC — перемонтируем конфиг Prometheus из ConfigMap; храним TSDB на PVC |
| 16 | Чарт `kube-prometheus-stack` с CRD `ServiceMonitor`, которые сами обнаруживают приложение — тот же PromQL, но инфраструктура Kubernetes |

---

**Good luck!** 🚀

**Удачи!** 🚀

> **Remember:** Counters need `rate()`. Percentiles need `histogram_quantile()` *outside* the `sum by (le)`. A label must never be unique per request or per user. **RED for every service. USE for every resource.** And `/metrics` should never count itself.
>
> **Помни:** счётчикам нужен `rate()`. Перцентилям нужен `histogram_quantile()` *снаружи* `sum by (le)`. Метка никогда не должна быть уникальной для каждого запроса или пользователя. **RED для каждого сервиса. USE для каждого ресурса.** И `/metrics` никогда не должен считать сам себя.
