# Lab 17 — Cloudflare Workers Edge Deployment

Лабораторная 17 — развёртывание на периферии Cloudflare Workers

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Edge%20Computing-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![type](https://img.shields.io/badge/type-Exam%20Alternative-purple)

> **Goal:** Build a serverless HTTP API on Cloudflare's global edge — a V8 isolate, not a container — and back it with your Lab 9 Kubernetes measurements in a real comparison.
> **Deliverable:** A PR from `lab17` with `edge-api/` (the Worker project) and `docs/LAB17.md` (the comparison + evidence).
>
> **Цель:** собрать бессерверный HTTP API на глобальной периферии Cloudflare — в V8-изоляте, а не в контейнере, — и подкрепить его измерениями Kubernetes из лабы 9 в реальном сравнении.
> **Что сдать:** PR из ветки `lab17` с каталогом `edge-api/` (проект Worker) и файлом `docs/LAB17.md` (сравнение и доказательства).

## Overview

Обзор

Cloudflare Workers runs your code in a **V8 isolate** — a sandboxed JS heap inside an already-running `workerd` process at 330+ points of presence. Cold start is **<5 ms**, the memory floor is ~3 MB per isolate, and there is no region to pick: every deploy is global. That is the opposite end of the abstraction spectrum from the Kubernetes you built across Labs 9–16.

Cloudflare Workers выполняет твой код в **V8-изоляте** — изолированной JS-куче внутри уже запущенного процесса `workerd` в более чем 330 точках присутствия. Холодный старт — **<5 мс**, нижняя граница памяти — около 3 МБ на изолят, и регион выбирать не нужно: каждый деплой глобальный. Это противоположный конец спектра абстракций по сравнению с Kubernetes, который ты строил в лабах 9–16.

In this lab you will practice:

В этой лабе ты отработаешь:

- Scaffolding a Worker with **Wrangler 4.95+** (4.x is **local-first** by default; `--remote` is opt-in)
  * Создание Worker через **Wrangler 4.95+** (в 4.x по умолчанию **local-first**; `--remote` включается отдельно)
- Writing the `export default { async fetch(...) }` handler shape from the V8 runtime
  * Написание обработчика в форме `export default { async fetch(...) }` для среды V8
- Wiring edge-attached state with **Workers KV** + Wrangler secrets
  * Связывание состояния на периферии через **Workers KV** и секреты Wrangler
- Reading edge request metadata (`request.cf.colo`) to prove the PoP routing model
  * Чтение метаданных запроса на периферии (`request.cf.colo`), чтобы доказать модель маршрутизации по PoP
- Producing an **honest, measured comparison** of Workers vs your Lab 9 K8s deployment
  * Получение **честного сравнения на измерениях** между Workers и твоим развёртыванием K8s из лабы 9

> ⚠️ **Scope:** this lab does **not** ship your Lab 2 Docker image. Workers has no `FROM`, no ingress, no HPA, no filesystem. You will build a Workers-native API that preserves the same operational concerns (routes, health, config, state, logs, deploys) in the edge model — and write up where that re-shapes the problem.
>
> **Область охвата:** эта лаба **не** разворачивает Docker-образ из лабы 2. В Workers нет `FROM`, нет ingress, нет HPA и нет файловой системы. Ты построишь нативный для Workers API, который сохраняет те же эксплуатационные задачи (маршруты, health, конфиг, состояние, логи, деплои) в периферийной модели, — и опишешь, как это меняет постановку задачи.

> ⚠️ **This is a bonus / Exam Alternative lab.** Complete both Lab 17 (12 pts) and Lab 18 (12 pts) to the required bar to replace the final exam.
>
> **Это бонусная лаба / альтернатива экзамену.** Выполни обе лабы — 17 (12 баллов) и 18 (12 баллов) — на требуемом уровне, чтобы заменить финальный экзамен.

---

## Exam Alternative Requirements

Требования для замены экзамена

| Requirement | Details |
|---|---|
| **Deadline** | 1 week before the exam date |
| **Minimum Score** | **10/12** on each of Lab 17 and Lab 18 |
| **Must Complete** | Both Lab 17 **and** Lab 18 |
| **Total Points** | Together they form the **exam replacement** option |

| Требование | Подробности |
|---|---|
| **Дедлайн** | За 1 неделю до даты экзамена |
| **Минимальный балл** | **10/12** за каждую из лаб 17 и 18 |
| **Что нужно сдать** | Обе лабы — 17 **и** 18 |
| **Всего баллов** | Вместе они образуют вариант **замены экзамена** |

> Taken on its own, Lab 17 is a **bonus** lab: **10 pts** of main tasks + a **2 pt** bonus task = **12 pts max**.
>
> Отдельно взятая лаба 17 — **бонусная**: **10 баллов** за основные задания плюс **2 балла** за бонус = **максимум 12 баллов**.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**У тебя должно быть с прошлых лаб:**

- **Lab 9** — your K8s Deployment + Service running on k3d, with the same HTTP service you've been growing since Lab 1. You'll re-use its cold-start, latency, and resource numbers in Task 3.
  * **Лаба 9** — твой Deployment + Service в K8s на k3d с тем же HTTP-сервисом, который растёт с лабы 1. Его цифры холодного старта, задержки и потребления ресурсов ты снова используешь в задании 3.

**This lab adds:**

**Эта лаба добавляет:**

- `edge-api/` — a Cloudflare Worker (one `src/index.js`, one `wrangler.toml`, one KV namespace)
  * `edge-api/` — Cloudflare Worker (один `src/index.js`, один `wrangler.toml`, один namespace KV)
- `docs/LAB17.md` — your submission report including the K8s-vs-Workers comparison
  * `docs/LAB17.md` — отчёт о сдаче, включая сравнение K8s и Workers

> **Regional connectivity note:** In some countries and networks (including parts of Russia), Cloudflare services may be partially restricted. If `npx wrangler whoami` or `npx wrangler deploy` fail with vague network errors, the problem may be your network path. If you use a VPN, prefer **full-tunnel** / global-routing mode — split-tunnel setups can let Node.js traffic bypass the VPN and hit the restricted network.
>
> **Замечание о региональной связности:** в некоторых странах и сетях (в том числе в части России) сервисы Cloudflare могут быть частично ограничены. Если `npx wrangler whoami` или `npx wrangler deploy` падают с невнятными сетевыми ошибками, проблема может быть в сетевом маршруте. Если используешь VPN, предпочитай **full-tunnel** / глобальную маршрутизацию — при split-tunnel трафик Node.js может обходить VPN и уходить в ограниченную сеть.

> **No Cloudflare account?** You can complete every task except the live `workers.dev` deploy and `request.cf` capture using `wrangler dev --local` (`workerd` runs the same V8 isolate runtime locally). See the **Documented-gap** sub-section under Task 1.6 for the substitute proof. The bonus task and full marks on Task 3 still require a real deploy.
>
> **Нет аккаунта Cloudflare?** Все задания, кроме живого деплоя на `workers.dev` и захвата `request.cf`, можно выполнить через `wrangler dev --local` (`workerd` запускает ту же среду V8-изолятов локально). Заменитель доказательства см. в подразделе **Documented-gap** в задании 1.6. Бонусное задание и полный балл за задание 3 всё равно требуют реального деплоя.

---

## Setup

Настройка

You need:

Тебе понадобится:

```bash
node --version           # 18+
npm --version
npx wrangler --version   # 4.95.0 or newer (4.x is local-first)
```

Пояснение: проверь версии Node.js (нужна 18+), npm и Wrangler (4.95.0 или новее).

If Wrangler isn't installed yet, you'll install it via the C3 scaffolder in Task 1.

Если Wrangler ещё не установлен, ты установишь его через скаффолдер C3 в задании 1.

---

## Task 1 — Scaffold, Build & Deploy a Worker API (4 pts)

Задание 1 — создание, сборка и деплой Worker API (4 балла)

### 1.1 — Scaffold the project

1.1 — Создание проекта

Use C3 (`create-cloudflare`) with the **Hello World / Worker only** template:

Используй C3 (`create-cloudflare`) с шаблоном **Hello World / Worker only**:

```bash
npm create cloudflare@latest -- edge-api
cd edge-api
```

Пояснение: команда создаёт проект Worker в каталоге `edge-api` и переходит в него.

When prompted, pick: *Hello World example* → *Worker only* → **JavaScript** (you may use TypeScript if you prefer; the skeleton below is JS) → Git: **Yes** → Deploy now: **No**.

При появлении вопросов выбери: *Hello World example* → *Worker only* → **JavaScript** (можно взять TypeScript, если хочешь; скелет ниже на JS) → Git: **Yes** → Deploy now: **No**.

Then:

Затем:

```bash
npx wrangler --version   # confirm 4.95 or newer
```

Пояснение: убедись, что установлен Wrangler 4.95 или новее.

> Why 4.95+? Wrangler 4.x switched `wrangler dev` to **local-first** — it runs your Worker in `workerd` on your machine. The `--remote` flag is now opt-in. If you're on a stale 3.x, `dev` hits Cloudflare's preview infra, which behaves differently (and costs requests).
>
> Почему 4.95+? В Wrangler 4.x команда `wrangler dev` стала **local-first** — она запускает твой Worker в `workerd` на твоей машине. Флаг `--remote` теперь включается отдельно. Если у тебя застарелый 3.x, `dev` обращается к preview-инфраструктуре Cloudflare, которая ведёт себя иначе (и тратит запросы).

### 1.2 — Configure `wrangler.toml`

1.2 — Настройка `wrangler.toml`

The C3 scaffolder generates a `wrangler.jsonc`. Either keep that or rename to `wrangler.toml` — both work; this lab uses TOML for brevity. Open it and fill in the values yourself:

Скаффолдер C3 генерирует `wrangler.jsonc`. Можешь оставить его или переименовать в `wrangler.toml` — работает и то и другое; в этой лабе для краткости используется TOML. Открой файл и сам заполни значения:

```toml
# wrangler.toml — YOUR TASK: fill every value
name = "___"                    # YOUR TASK: pick a worker name (lowercase, hyphens, your-handle-ish)
main = "___"                    # YOUR TASK: path to your entry file (e.g. src/index.js)
compatibility_date = "___"      # YOUR TASK: ISO date — pin runtime behavior; pick today (YYYY-MM-DD)

# Plaintext vars (Task 2 will add real ones)
[vars]
APP_NAME = "edge-api"
```

Пояснение: заполни `name`, `main` и `compatibility_date`; в `[vars]` пока лежит только открытая переменная `APP_NAME`.

> **Why `compatibility_date`?** It pins the `workerd` runtime semantics so Cloudflare can ship new defaults without breaking your Worker. Picking today's date is fine; never leave it blank.
>
> **Зачем `compatibility_date`?** Она фиксирует семантику среды `workerd`, чтобы Cloudflare могла вводить новые значения по умолчанию, не ломая твой Worker. Указать сегодняшнюю дату нормально; никогда не оставляй поле пустым.

### 1.3 — Write the `fetch` handler

1.3 — Пишем обработчик `fetch`

Open `src/index.js`. The C3 scaffold gives you a "Hello World" — **delete it**. The Worker entry point is a single object exported as `default`, with one async `fetch(request, env, ctx)` method. That's the entire shape:

Открой `src/index.js`. Скаффолдер C3 даёт «Hello World» — **удали его**. Точка входа Worker — единственный объект, экспортируемый как `default`, с одним асинхронным методом `fetch(request, env, ctx)`. Вот и вся форма:

```javascript
// src/index.js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // YOUR TASK: route on url.pathname for "/", "/health", "/echo"
    // YOUR TASK: GET /         → Response.json with service info
    //            keys (mirror Lab 1 shape): { service, version, framework, runtime, edge: { colo }, path }
    // YOUR TASK: GET /health   → Response.json({ status: ___, edge: true }) with HTTP 200
    // YOUR TASK: /echo
    //            POST → echo the request body back (same Content-Type)
    //            GET  → 405 Method Not Allowed, JSON body { error: ___, allow: ___ }
    // YOUR TASK: anything else → 404 with JSON body { error: ___, path: url.pathname }

    return ___;
  }
};
```

Пояснение: реализуй маршрутизацию по `url.pathname`; для `/echo` POST возвращает тело запроса обратно, а GET — 405.

Requirements:

Требования:

- **`GET /`** returns five top-level keys minimum: `service`, `version`, `framework` (`"cloudflare-workers"`), `runtime` (`"v8-isolate"`), `edge.colo` (read from `request.cf?.colo` — see note below), and `path`. Keep it shaped like Lab 1's `/` JSON so the contrast is fair.
  * **`GET /`** возвращает минимум пять ключей верхнего уровня: `service`, `version`, `framework` (`"cloudflare-workers"`), `runtime` (`"v8-isolate"`), `edge.colo` (читается из `request.cf?.colo` — см. примечание ниже) и `path`. Держи форму как у JSON `/` из лабы 1, чтобы контраст был честным.
- **`GET /health`** returns `{ status: "ok", edge: true }` with HTTP 200. This is what `wrangler tail` will help you observe in Task 3.
  * **`GET /health`** возвращает `{ status: "ok", edge: true }` с HTTP 200. Именно это `wrangler tail` поможет наблюдать в задании 3.
- **`POST /echo`** reads the request body (`await request.text()` or `await request.json()`) and returns it. `GET /echo` returns **405** with `Allow: POST` semantics in the JSON.
  * **`POST /echo`** читает тело запроса (`await request.text()` или `await request.json()`) и возвращает его. `GET /echo` возвращает **405** с семантикой `Allow: POST` в JSON.
- **Anything else** returns **404** with a JSON body (never HTML — same lesson as Lab 1).
  * **Всё остальное** возвращает **404** с JSON-телом (никогда не HTML — тот же урок, что в лабе 1).

> **`request.cf?.colo`** is populated by Cloudflare at the real edge. In `wrangler dev --local` it's `undefined` or a stub (`"TBS"`). That's expected; your `/` handler should use optional chaining so it doesn't throw.
>
> **`request.cf?.colo`** заполняется Cloudflare на реальной периферии. В `wrangler dev --local` это `undefined` или заглушка (`"TBS"`). Это ожидаемо; обработчик `/` должен использовать опциональную цепочку, чтобы не падать.

> **`Response.json(obj, init?)`** is the modern way to return JSON from a Worker — sets `Content-Type: application/json` for you. Available since `compatibility_date: 2022-01-31`. If you set an older date you'd have to do `new Response(JSON.stringify(obj), { headers: { "content-type": "application/json" } })` by hand — another reason to pin a recent date in 1.2.
>
> **`Response.json(obj, init?)`** — современный способ вернуть JSON из Worker: `Content-Type: application/json` выставляется автоматически. Доступен начиная с `compatibility_date: 2022-01-31`. При более старой дате пришлось бы вручную писать `new Response(JSON.stringify(obj), { headers: { "content-type": "application/json" } })` — ещё одна причина зафиксировать свежую дату в 1.2.

### 1.4 — Run locally on `workerd`

1.4 — Локальный запуск на `workerd`

```bash
npx wrangler dev
```

Пояснение: запускает локальный `workerd` на `http://localhost:8787`.

Wrangler boots a `workerd` process on `http://localhost:8787`. This is the **same** V8-isolate runtime Cloudflare runs at the edge — local dev is faithful to production behavior (modulo `request.cf` and global routing).

Wrangler поднимает процесс `workerd` на `http://localhost:8787`. Это **та же** среда V8-изолятов, которую Cloudflare запускает на периферии, — локальная разработка достоверно повторяет продакшен (с поправкой на `request.cf` и глобальную маршрутизацию).

`YOUR TASK`: hit every route with `curl` and capture the JSON. Suggested checks:

`YOUR TASK`: пройди каждый маршрут через `curl` и зафиксируй JSON. Предлагаемые проверки:

- `curl -s http://localhost:8787/ | jq .`
  * Проверка корневого маршрута.
- `curl -s http://localhost:8787/health | jq .`
  * Проверка маршрута health.
- `curl -s -X POST -H 'content-type: application/json' -d '{"ping":42}' http://localhost:8787/echo | jq .`
  * POST-запрос на `/echo` с JSON-телом.
- `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8787/echo` (GET → expect `HTTP 405`)
  * GET на `/echo` — ожидается `HTTP 405`.
- `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8787/nope` (expect `HTTP 404`)
  * Запрос на несуществующий маршрут — ожидается `HTTP 404`.

### 1.5 — Deploy globally

1.5 — Глобальный деплой

```bash
npx wrangler login         # OAuth in browser; opens a localhost callback
npx wrangler whoami        # confirms account + lists your workers.dev subdomain
npx wrangler deploy
```

Пояснение: вход в аккаунт, проверка аккаунта и сабдомена `workers.dev`, затем глобальный деплой.

> ⚠️ **Headless / remote-VM gotcha:** `wrangler login` opens a browser locally and listens on `127.0.0.1:8976` for the OAuth callback. If you're SSH'd into a remote VM with no local browser, the callback never reaches your browser. Two options: (a) `ssh -L 8976:localhost:8976 <user@host>` to port-forward the OAuth callback to your laptop, or (b) export `CLOUDFLARE_API_TOKEN=<token>` from a token you create in the dashboard (Account → API Tokens → "Edit Cloudflare Workers" template). Document the path you took in §1.6.
>
> **Ловушка при работе без графической сессии / на удалённой ВМ:** `wrangler login` открывает браузер локально и слушает `127.0.0.1:8976` для OAuth-колбэка. Если ты подключён по SSH к удалённой ВМ без браузера, колбэк до браузера не дойдёт. Два варианта: (а) `ssh -L 8976:localhost:8976 <user@host>`, чтобы пробросить OAuth-колбэк на ноутбук, или (б) экспортировать `CLOUDFLARE_API_TOKEN=<token>` из токена, созданного в панели (Account → API Tokens → шаблон «Edit Cloudflare Workers»). Опиши выбранный путь в §1.6.

The output prints `https://<worker-name>.<your-subdomain>.workers.dev`. Hit it with `curl` and confirm the same JSON — except `edge.colo` is now a **real** PoP code (`"FRA"`, `"WAW"`, `"SVO"`, ...).

В выводе появится `https://<worker-name>.<your-subdomain>.workers.dev`. Обратись к нему через `curl` и убедись, что JSON тот же, — только `edge.colo` теперь **настоящий** код PoP (`"FRA"`, `"WAW"`, `"SVO"`, ...).

### 1.6 — Proof of work

1.6 — Подтверждение работы

**Paste into `docs/LAB17.md`:**

**Вставь в `docs/LAB17.md`:**

- `npx wrangler --version` output (must be 4.95+)
  * Вывод `npx wrangler --version` (должен быть 4.95+)
- All five `curl` captures from 1.4 against `localhost:8787` (with the **exact commands** you ran)
  * Все пять захватов `curl` из 1.4 против `localhost:8787` (с **точными командами**, которые ты выполнял)
- Either:
  * Одно из двух:
  - **(a)** `curl` capture of `GET /` against your deployed `workers.dev` URL showing a **real** `edge.colo`, *or*
    * **(а)** захват `curl` для `GET /` против развёрнутого URL `workers.dev`, показывающий **настоящий** `edge.colo`, *или*
  - **(b)** **Documented-gap** sub-section: explain that you have no CF account (or are on a network where deploy is blocked), then paste the **local** `GET /` capture and write 2–3 sentences on what would be different at the real edge (`colo` populated, latency under 50 ms in your region, anycast routing).
    * **(б)** подраздел **Documented-gap**: объясни, что у тебя нет аккаунта CF (или ты в сети, где деплой заблокирован), затем вставь **локальный** захват `GET /` и напиши 2–3 предложения о том, что отличалось бы на реальной периферии (заполненный `colo`, задержка меньше 50 мс в твоём регионе, anycast-маршрутизация).

---

## Task 2 — Config, Secrets & Edge State (3 pts)

Задание 2 — конфигурация, секреты и состояние на периферии (3 балла)

### 2.1 — Why this matters

2.1 — Почему это важно

Your Worker now responds. The next test is whether you can give it **configuration** (vars), **credentials** (secrets), and **state** (KV) without ever putting them in Git. This is the same separation-of-concerns problem you solved in Labs 1 (env vars), 11 (OpenBao secrets), and 12 (ConfigMaps + PVCs) — the primitives are just edge-shaped.

Твой Worker уже отвечает. Следующая проверка — сумеешь ли ты дать ему **конфигурацию** (переменные), **учётные данные** (секреты) и **состояние** (KV), ни разу не положив их в Git. Это та же задача разделения ответственности, которую ты решал в лабах 1 (переменные окружения), 11 (секреты OpenBao) и 12 (ConfigMaps + PVC), — просто примитивы здесь имеют периферийную форму.

### 2.2 — Plaintext variable

2.2 — Открытая переменная

You already added `APP_NAME` to `wrangler.toml` in 1.2. `YOUR TASK`: read it inside the Worker (`env.APP_NAME`) and surface it in your `GET /` `service` field. In one sentence in `docs/LAB17.md`, say why vars are **unsuitable for secrets** (hint: they're in the dashboard + Git + readable by any collaborator).

Ты уже добавил `APP_NAME` в `wrangler.toml` в 1.2. `YOUR TASK`: прочитай её внутри Worker (`env.APP_NAME`) и выведи в поле `service` ответа `GET /`. Одним предложением в `docs/LAB17.md` объясни, почему переменные **не годятся для секретов** (подсказка: они видны в панели, в Git и любому коллаборатору).

### 2.3 — Wrangler secrets

2.3 — Секреты Wrangler

Create **two** secrets. Wrangler prompts you for the value — it never lands in your repo:

Создай **два** секрета. Wrangler спросит значение — оно никогда не попадёт в репозиторий:

```bash
npx wrangler secret put ___        # YOUR TASK: name your first secret
npx wrangler secret put ___        # YOUR TASK: name your second secret
npx wrangler secret list           # names only, no values
```

Пояснение: команды создают секреты; значения вводятся интерактивно и не попадают в репозиторий.

Then read them inside the Worker. The bindings appear on `env`:

Затем прочитай их внутри Worker. Биндинги появляются на `env`:

```javascript
// YOUR TASK: in your fetch handler, read the secret you put above
const token = env.___;             // YOUR TASK: must match the name in `wrangler secret put`
```

Пояснение: секреты доступны как свойства объекта `env`.

Surface one of them in a **gated** route — for example, `GET /admin` returns 401 unless `request.headers.get("authorization") === \`Bearer ${env.YOUR_SECRET}\``. The point is to prove the secret round-trips from `wrangler secret put` → `env` → request handling.

Выведи один из них в **защищённом** маршруте — например, `GET /admin` возвращает 401, если `request.headers.get("authorization")` не равен `Bearer ${env.YOUR_SECRET}`. Смысл — доказать, что секрет проходит путь от `wrangler secret put` → `env` → обработка запроса.

> **Local dev + secrets:** `wrangler dev` reads local-only overrides from `.dev.vars` (a gitignored file shaped like `.env`). For the real deploy, only `wrangler secret put` counts.
>
> **Локальная разработка и секреты:** `wrangler dev` читает локальные переопределения из `.dev.vars` (файл в `.gitignore`, по форме как `.env`). Для реального деплоя считается только `wrangler secret put`.

### 2.4 — Workers KV (mandatory)

2.4 — Workers KV (обязательно)

KV is Workers' eventually-consistent key/value store — the right primitive for flags, counters, and small config blobs. Two-step setup: create the namespace, then bind it.

KV — это eventually-consistent хранилище ключ/значение в Workers, правильный примитив для флагов, счётчиков и небольших конфигов. Настройка в два шага: создать namespace, затем привязать его.

```bash
npx wrangler kv namespace create ___      # YOUR TASK: pick a binding name (e.g. SETTINGS)
```

Пояснение: команда создаёт namespace KV и печатает фрагмент конфигурации.

The command prints a config snippet. Copy the **id** it gives you into `wrangler.toml`:

Команда печатает фрагмент конфига. Скопируй **id**, который она выдаст, в `wrangler.toml`:

```toml
[[kv_namespaces]]
binding = "___"     # YOUR TASK: same binding name you used above — this is what env.<binding> exposes
id      = "___"     # YOUR TASK: the namespace id Wrangler just printed (a 32-char hex string)
```

Пояснение: блок `[[kv_namespaces]]` связывает namespace с биндингом `env.<binding>`.

Then write a `/counter` route that increments a KV-backed visit counter:

Затем напиши маршрут `/counter`, который увеличивает счётчик посещений в KV:

```javascript
// YOUR TASK: inside fetch, handle url.pathname === "/counter"
// 1. read the current value with env.<binding>.get("visits") (returns string or null)
// 2. compute the next integer
// 3. write it back with env.<binding>.put("visits", String(n))
// 4. return Response.json({ visits: n })
```

Пояснение: обработчик `/counter` читает значение `visits`, вычисляет следующее целое и записывает его обратно.

Round-trip from the CLI to convince yourself the namespace is real:

Сделай round-trip из CLI, чтобы убедиться, что namespace реальный:

```bash
npx wrangler kv key put --binding=___ "visits" "0"        # YOUR TASK: binding name
npx wrangler kv key get --binding=___ "visits"            # YOUR TASK: same binding
```

Пояснение: проверка существования namespace — запись и чтение ключа `visits`.

> **D1, R2, Durable Objects** all live on the same `env.<binding>` model:
> - **D1** — SQLite per region, replicated. Use when you need joins.
> - **R2** — S3-compatible blob storage, **no egress fees**. Use for assets/backups.
> - **Durable Objects** — strongly-consistent stateful actors. Use for chat rooms, leaderboards, rate-limiters.
> KV is the right pick here; the Bonus Task lets you swap in one of the others.
>
> **D1, R2 и Durable Objects** живут на одной модели `env.<binding>`:
> - **D1** — SQLite по регионам, с репликацией. Используй, когда нужны джойны.
> - **R2** — S3-совместимое хранилище блобов, **без платы за egress**. Для ассетов и бэкапов.
> - **Durable Objects** — строго согласованные акторы с состоянием. Для чатов, лидербордов и ограничителей частоты запросов.
> KV здесь — правильный выбор; в бонусном задании можно заменить его на один из них.

### 2.5 — Proof of work

2.5 — Подтверждение работы

**Paste into `docs/LAB17.md`:**

**Вставь в `docs/LAB17.md`:**

- `npx wrangler secret list` output showing your two secret **names** (values must not appear)
  * Вывод `npx wrangler secret list` с **именами** двух твоих секретов (значения появляться не должны)
- A `curl` proving the gated route from 2.3 — one call without the header (401), one with (200)
  * `curl`, доказывающий защищённый маршрут из 2.3: один вызов без заголовка (401), один с ним (200)
- `npx wrangler kv key put` / `kv key get` round-trip from 2.4 (illustrative IDs are fine; your IDs will differ)
  * Round-trip `npx wrangler kv key put` / `kv key get` из 2.4 (иллюстративные ID подойдут; твои ID будут другими)
- A `curl http://localhost:8787/counter` (or your deployed URL) hit **three times** — the third response must show `visits: 3` *and* must continue, not reset, across a redeploy if you deployed
  * `curl http://localhost:8787/counter` (или твой развёрнутый URL) **три раза** — третий ответ должен показать `visits: 3` *и* продолжить счёт, а не сбросить его после повторного деплоя, если ты деплоил
- The one-sentence reason vars are unsuitable for secrets
  * Причина в одном предложении, почему переменные не годятся для секретов

---

## Task 3 — Observability, Ops & K8s Comparison (3 pts)

Задание 3 — наблюдаемость, эксплуатация и сравнение с K8s (3 балла)

This is the lecture-16 payoff. Workers and K8s solve the same problem (run my HTTP code in production); they pay very different prices.

Вот где пригодится лекция 16. Workers и K8s решают одну задачу (запустить мой HTTP-код в продакшене), но платят очень разную цену.

### 3.1 — Live logs

3.1 — Живые логи

Add at least one `console.log()` call in your `fetch` handler — log the path and the (possibly undefined) colo:

Добавь хотя бы один вызов `console.log()` в обработчик `fetch` — логируй путь и (возможно, undefined) colo:

```javascript
// YOUR TASK: add console.log with at least the path and request.cf?.colo
console.log(___);
```

Пояснение: добавь `console.log`, чтобы видеть путь и colo в логах.

Then in another terminal:

Затем в другом терминале:

```bash
npx wrangler tail
```

Пояснение: `wrangler tail` стримит логи Worker почти в реальном времени.

Hit any route from a third shell. The log line streams from the edge in near real-time.

Обратись к любому маршруту из третьего терминала. Строка лога прилетает с периферии почти в реальном времени.

`YOUR TASK`: capture **one** `wrangler tail` line — paste the literal log entry (timestamps and all) into `docs/LAB17.md`.

`YOUR TASK`: зафиксируй **одну** строку `wrangler tail` — вставь буквальную запись лога (вместе с временными метками) в `docs/LAB17.md`.

### 3.2 — Versions and rollback

3.2 — Версии и откат

Deploy at least **two** versions of your Worker. Trivial change between them is fine (bump the `version` field in `GET /`). Then list and roll back:

Задеплой как минимум **две** версии Worker. Незначительное изменение между ними подойдёт (поменяй поле `version` в `GET /`). Затем выведи список и откатись:

```bash
npx wrangler deployments list      # YOUR TASK: capture this — version IDs + timestamps
npx wrangler rollback              # YOUR TASK: capture this — confirms the rollback
```

Пояснение: список деплоев с ID версий и временем, затем откат к предыдущей версии.

If you have no CF account, document this as a gap: explain what each command would print (Wrangler shows a numbered list of versions with author + timestamp; `rollback` prompts confirmation then restores the previous version atomically).

Если аккаунта CF нет, опиши это как пробел: объясни, что вывела бы каждая команда (Wrangler показывает нумерованный список версий с автором и временем; `rollback` запрашивает подтверждение и атомарно восстанавливает предыдущую версию).

### 3.3 — The K8s vs Workers table (the whole point)

3.3 — Таблица K8s против Workers (в этом вся суть)

Pull up your **Lab 9** notes: cold start of a fresh pod, request latency from your laptop to the k3d Service, the resources you reserved per pod. If you've forgotten, re-run `kubectl get pods -w` against a fresh Deployment and `time curl ...` once it's Ready. Then fill the table **with your own measurements** — not the lecture's defaults:

Открой свои заметки по **лабе 9**: холодный старт свежего пода, задержка запроса с твоего ноутбука до Service в k3d, ресурсы, зарезервированные на под. Если забыл, перезапусти `kubectl get pods -w` на свежем Deployment и `time curl ...`, когда он будет Ready. Затем заполни таблицу **своими измерениями**, а не значениями по умолчанию из лекции:

`YOUR TASK`: fill every cell of this table in `docs/LAB17.md`. Cite where each number came from.

`YOUR TASK`: заполни каждую ячейку этой таблицы в `docs/LAB17.md`. Укажи, откуда взялась каждая цифра.

| Aspect | Kubernetes (Lab 9, your numbers) | Cloudflare Workers (this lab, your numbers) |
|---|---|---|
| Cold start (first request after scale-from-zero) | ___ s (`kubectl get pods -w` from `Pending` → `Ready` + first `200`) | ___ ms (`time curl` against fresh deploy) |
| Memory floor per instance | ___ Mi (your pod `resources.requests.memory`) | ~3 MB (V8 isolate, documented) |
| Geographic distribution | ___ (your single k3d node — what region/laptop) | 330+ PoPs, all of them automatically |
| Cost per 1M requests at idle | ___ ($/mo at 0 traffic — your node bill if rented; $0 on k3d, but say what a managed equivalent would cost) | **Free tier:** 100k req/day × 30 = ~3M/mo at $0. **Paid ($5/mo):** 10M req + 50 ms CPU |
| State primitive used | ___ (e.g. PVC + local-path from Lab 12) | Workers KV (this lab) |
| Long-running task support | yes — any process | 30 s wall-clock max (paid), CPU 10 ms (free) / 50 ms (paid, burstable to 5 min) |
| Best use case | ___ (your one-line take) | ___ (your one-line take) |

| Аспект | Kubernetes (лаба 9, твои цифры) | Cloudflare Workers (эта лаба, твои цифры) |
|---|---|---|
| Холодный старт (первый запрос после масштабирования с нуля) | ___ с (`kubectl get pods -w` от `Pending` → `Ready` + первый `200`) | ___ мс (`time curl` к свежему деплою) |
| Нижняя граница памяти на инстанс | ___ МиБ (`resources.requests.memory` твоего пода) | ~3 МБ (V8-изолят, по документации) |
| Географическое распределение | ___ (твой единственный узел k3d — какой регион или ноутбук) | 330+ PoP, все автоматически |
| Стоимость 1 млн запросов при простое | ___ ($/мес при нулевом трафике — счёт за узел, если он арендован; на k3d $0, но укажи, сколько стоило бы managed-решение) | **Бесплатный тариф:** 100 тыс. запросов/день × 30 = ~3 млн/мес за $0. **Платный ($5/мес):** 10 млн запросов + 50 мс CPU |
| Используемый примитив состояния | ___ (например, PVC + local-path из лабы 12) | Workers KV (эта лаба) |
| Поддержка долгих задач | да — любой процесс | максимум 30 с реального времени (платно), CPU 10 мс (бесплатно) / 50 мс (платно, бёрст до 5 мин) |
| Лучший сценарий применения | ___ (твой вывод в одну строку) | ___ (твой вывод в одну строку) |

> **The point:** Workers wins on cold start and price floor for spiky, latency-sensitive APIs. K8s wins on long-running, stateful, native, multi-service workloads. The numbers from your own labs are what make the comparison honest.
>
> **Суть:** Workers выигрывает по холодному старту и нижней цене для пиковых API, чувствительных к задержке. K8s выигрывает на долгоживущих, stateful, нативных и многосервисных нагрузках. Именно цифры из твоих лаб делают сравнение честным.

### 3.4 — Reflection (3 sentences, in `docs/LAB17.md`)

3.4 — Рефлексия (3 предложения в `docs/LAB17.md`)

`YOUR TASK`: answer in your own words —

`YOUR TASK`: ответь своими словами —

1. What was **easier** in Workers than in your K8s setup from Lab 9?
   * Что было **проще** в Workers, чем в твоей K8s-конфигурации из лабы 9?
2. What felt **more constrained**? (V8 isolate has no `fs`, no `Buffer` by default, no native binaries, 10 ms CPU on free tier.)
   * Что ощущалось **более ограниченным**? (В V8-изоляте нет `fs`, по умолчанию нет `Buffer`, нет нативных бинарников, 10 мс CPU на бесплатном тарифе.)
3. What changed because Workers is **not a Docker host**? (Think: no `Dockerfile`, no scan, no ingress, no HPA — but also no choice about base image, no Lab 2 multi-stage trick to fall back on.)
   * Что изменилось из-за того, что Workers — **не Docker-хост**? (Подумай: нет `Dockerfile`, нет сканирования, нет ingress, нет HPA — но и нет выбора базового образа, нет многоступенчатого трюка из лабы 2, на который можно опереться.)

### 3.5 — Proof of work

3.5 — Подтверждение работы

**Paste into `docs/LAB17.md`:**

**Вставь в `docs/LAB17.md`:**

- The `wrangler tail` line from 3.1
  * Строка `wrangler tail` из 3.1
- The `deployments list` + `rollback` captures from 3.2 (or the documented-gap equivalent)
  * Захваты `deployments list` + `rollback` из 3.2 (или эквивалент documented-gap)
- The fully populated table from 3.3 with your own measured numbers
  * Полностью заполненная таблица из 3.3 с твоими собственными измеренными цифрами
- The three-sentence reflection from 3.4
  * Рефлексия из трёх предложений из 3.4

---

## Bonus Task — Custom Domain or D1/R2/Durable Objects (2 pts)

Бонусное задание — свой домен или D1/R2/Durable Objects (2 балла)

Pick **one**:

Выбери **один** вариант:

**Option A — Routing beyond `workers.dev`.** Attach your Worker to a Custom Domain you control (a zone on Cloudflare). If you have no domain, write the full procedure — what `wrangler.toml` looks like, what DNS records Cloudflare creates, what cert is provisioned — and show the `routes` block:

**Вариант A — маршрутизация за пределами `workers.dev`.** Привяжи Worker к своему Custom Domain (зоне в Cloudflare). Если домена нет, опиши полную процедуру — как выглядит `wrangler.toml`, какие DNS-записи создаёт Cloudflare, какой сертификат выпускается — и покажи блок `routes`:

```toml
# YOUR TASK: routes block for a custom domain you control (or document precisely)
[[routes]]
pattern        = "___"            # YOUR TASK: e.g. api.example.com
custom_domain  = true
```

Пояснение: блок `[[routes]]` привязывает Worker к собственному домену.

**Option B — A second state primitive.** Add **D1**, **R2**, or a **Durable Object**, bind it, and use it from one route.

**Вариант B — второй примитив состояния.** Добавь **D1**, **R2** или **Durable Object**, привяжи его и используй в одном маршруте.

```bash
# YOUR TASK: pick ONE
npx wrangler d1 create ___        # SQLite per region
npx wrangler r2 bucket create ___ # blob storage, no egress fees
# Durable Objects need a class export — see the docs
```

Пояснение: команды создают один из примитивов состояния — D1, R2 или Durable Object.

Then bind in `wrangler.toml` and call the binding from a new route. In `docs/LAB17.md`, write **one paragraph** justifying why you'd pick it over KV for the use case you implemented (consistency? joins? blobs? coordination?).

Затем привяжи его в `wrangler.toml` и вызови биндинг из нового маршрута. В `docs/LAB17.md` напиши **один абзац** с обоснованием, почему для твоего сценария ты выбрал его, а не KV (согласованность? джойны? блобы? координация?).

---

## How to Submit

Как сдать

```bash
git switch -c lab17
git add edge-api/ docs/LAB17.md
git commit -m "feat(lab17): cloudflare worker edge API + K8s comparison"
git push -u origin lab17
```

Пояснение: создай ветку `lab17`, закоммить Worker и отчёт и запушь её.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab17` → `course-repo:master` *(reviewed)*
  * `your-fork:lab17` → `course-repo:master` *(с ревью)*
- `your-fork:lab17` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab17` → `your-fork:master` *(смержится в твой master, когда закончишь)*

Include the live `workers.dev` URL in the PR description if you deployed; otherwise reference the documented-gap section in `docs/LAB17.md`. **Never commit secret values.**

Укажи живой URL `workers.dev` в описании PR, если ты деплоил; иначе сошлись на раздел documented-gap в `docs/LAB17.md`. **Никогда не коммить значения секретов.**

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — scaffold, fetch handler, 3 routes + 404 + 405, local run, (deploy or documented gap)
- [ ] Task 2 done — var, ≥2 secrets, KV namespace + counter route, round-trip CLI
- [ ] Task 3 done — wrangler tail capture, deployments list + rollback, K8s-vs-Workers table filled, reflection
- [ ] Bonus done — Custom Domain config OR D1/R2/Durable Object wired (with justification)
```

Пояснение: чек-лист PR — отметь выполненные задания 1–3 и бонус.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (4 pts)

Задание 1 (4 балла)

- ✅ Wrangler 4.95+ installed; `whoami` works (if account) or gap documented
  * Установлен Wrangler 4.95+; `whoami` работает (если есть аккаунт) или пробел задокументирован
- ✅ `wrangler.toml` has `name`, `main`, `compatibility_date` filled with student-picked values
  * В `wrangler.toml` заполнены `name`, `main`, `compatibility_date` — значениями, выбранными студентом
- ✅ `fetch` handler implements `/`, `/health`, `/echo` (GET 405, POST echo), and JSON 404
  * Обработчик `fetch` реализует `/`, `/health`, `/echo` (GET 405, POST echo) и JSON 404
- ✅ All five local `curl` captures present in `docs/LAB17.md`
  * Все пять локальных захватов `curl` присутствуют в `docs/LAB17.md`
- ✅ Deploy capture against `workers.dev` (with real `edge.colo`) OR a documented-gap section
  * Захват деплоя против `workers.dev` (с настоящим `edge.colo`) **ИЛИ** раздел documented-gap

### Task 2 (3 pts)

Задание 2 (3 балла)

- ✅ `wrangler.toml` has `[vars]` plus a `[[kv_namespaces]]` block with student-filled `binding` + real `id`
  * В `wrangler.toml` есть `[vars]` и блок `[[kv_namespaces]]` с заполненными студентом `binding` и настоящим `id`
- ✅ `wrangler secret list` capture shows two named secrets (values not committed)
  * Захват `wrangler secret list` показывает два именованных секрета (значения не закоммичены)
- ✅ Gated route returns 401 without auth and 200 with the right header
  * Защищённый маршрут возвращает 401 без авторизации и 200 с правильным заголовком
- ✅ `/counter` increments across three calls; KV CLI round-trip captured
  * `/counter` увеличивается за три вызова; round-trip KV CLI зафиксирован

### Task 3 (3 pts)

Задание 3 (3 балла)

- ✅ `console.log()` present in the handler; one literal `wrangler tail` line in `docs/LAB17.md`
  * `console.log()` присутствует в обработчике; одна буквальная строка `wrangler tail` в `docs/LAB17.md`
- ✅ `deployments list` + `rollback` capture (or gap documented)
  * Захват `deployments list` + `rollback` (или задокументированный пробел)
- ✅ K8s-vs-Workers table fully populated with **the student's own Lab 9 measurements** + the student's Worker numbers
  * Таблица K8s против Workers полностью заполнена **собственными измерениями студента из лабы 9** и цифрами его Worker
- ✅ Three-sentence reflection answering all three prompts
  * Рефлексия из трёх предложений, отвечающая на все три вопроса

### Bonus Task (2 pts)

Бонусное задание (2 балла)

- ✅ Custom Domain config working or precisely documented, OR
  * Конфигурация Custom Domain работает или точно задокументирована, **ИЛИ**
- ✅ D1 / R2 / Durable Object created, bound, and used from a route — with a one-paragraph justification
  * D1 / R2 / Durable Object созданы, привязаны и использованы из маршрута — с обоснованием в один абзац

---

## Rubric

Таблица оценивания

| Task | Points | Criteria |
|---|---:|---|
| **Task 1** — Scaffold, build, deploy | **4** | Handler + routes correct, local run captured, deploy or documented gap |
| **Task 2** — Config, secrets, KV | **3** | Var + 2 secrets + KV counter all working, no values leaked |
| **Task 3** — Observability + comparison | **3** | Tail capture, deployments/rollback, **student-measured** comparison table, reflection |
| **Bonus** — Custom Domain / second state primitive | **2** | One option implemented or precisely documented |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|---|---:|---|
| **Задание 1** — создание, сборка, деплой | **4** | Обработчик и маршруты верны, локальный запуск зафиксирован, деплой или задокументированный пробел |
| **Задание 2** — конфиг, секреты, KV | **3** | Переменная + 2 секрета + KV-счётчик работают, значения не утекли |
| **Задание 3** — наблюдаемость + сравнение | **3** | Захват tail, деплои/откат, таблица сравнения **с измерениями студента**, рефлексия |
| **Бонус** — свой домен / второй примитив состояния | **2** | Один вариант реализован или точно задокументирован |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading:**

**Оценивание:**

- **10/10 main:** Worker deployed (or gap clearly justified), KV persistence shown, comparison table backed by real numbers, reflection is concrete
  * **10/10 основных:** Worker задеплоен (или пробел ясно обоснован), показана персистентность KV, таблица сравнения подкреплена реальными цифрами, рефлексия конкретна
- **8–9/10:** Working Worker, minor gaps in evidence or analysis
  * **8–9/10:** рабочий Worker, небольшие пробелы в доказательствах или анализе
- **6–7/10:** Basic deploy works but missing KV, observability, or comparison depth
  * **6–7/10:** базовый деплой работает, но не хватает KV, наблюдаемости или глубины сравнения
- **<6/10:** Incomplete implementation
  * **<6/10:** неполная реализация

---

## Resources

Ресурсы

<details>
<summary>📚 Core Cloudflare Workers Docs</summary>

Основная документация Cloudflare Workers

- [Cloudflare Workers overview](https://developers.cloudflare.com/workers/)
  * [Обзор Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Get started with Wrangler](https://developers.cloudflare.com/workers/get-started/guide/)
  * [Начало работы с Wrangler](https://developers.cloudflare.com/workers/get-started/guide/)
- [Wrangler v3 → v4 migration (local-by-default)](https://developers.cloudflare.com/workers/wrangler/migration/update-v3-to-v4/)
  * [Миграция Wrangler v3 → v4 (local-by-default)](https://developers.cloudflare.com/workers/wrangler/migration/update-v3-to-v4/)
- [Wrangler commands](https://developers.cloudflare.com/workers/wrangler/commands/)
  * [Команды Wrangler](https://developers.cloudflare.com/workers/wrangler/commands/)
- [Workers pricing (free tier: 100k req/day)](https://developers.cloudflare.com/workers/platform/pricing/)
  * [Цены Workers (бесплатный тариф: 100 тыс. запросов/день)](https://developers.cloudflare.com/workers/platform/pricing/)
- [How Workers works (V8 isolates)](https://developers.cloudflare.com/workers/reference/how-workers-works/)
  * [Как работают Workers (V8-изоляты)](https://developers.cloudflare.com/workers/reference/how-workers-works/)

</details>

<details>
<summary>🔐 Config, Secrets & State</summary>

Конфигурация, секреты и состояние

- [Environment variables](https://developers.cloudflare.com/workers/configuration/environment-variables/)
  * [Переменные окружения](https://developers.cloudflare.com/workers/configuration/environment-variables/)
- [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
  * [Секреты](https://developers.cloudflare.com/workers/configuration/secrets/)
- [Workers KV](https://developers.cloudflare.com/kv/) · [D1](https://developers.cloudflare.com/d1/) · [R2](https://developers.cloudflare.com/r2/) · [Durable Objects](https://developers.cloudflare.com/durable-objects/)
  * [Workers KV](https://developers.cloudflare.com/kv/) · [D1](https://developers.cloudflare.com/d1/) · [R2](https://developers.cloudflare.com/r2/) · [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Choosing a storage product](https://developers.cloudflare.com/workers/platform/storage-options/)
  * [Как выбрать продукт хранения](https://developers.cloudflare.com/workers/platform/storage-options/)
- [Request API and `request.cf`](https://developers.cloudflare.com/workers/runtime-apis/request/)
  * [Request API и `request.cf`](https://developers.cloudflare.com/workers/runtime-apis/request/)

</details>

<details>
<summary>📊 Observability & Deployments</summary>

Наблюдаемость и деплои

- [Observability overview](https://developers.cloudflare.com/workers/observability/)
  * [Обзор наблюдаемости](https://developers.cloudflare.com/workers/observability/)
- [`wrangler tail`](https://developers.cloudflare.com/workers/observability/logs/real-time-logs/)
  * [`wrangler tail`](https://developers.cloudflare.com/workers/observability/logs/real-time-logs/)
- [Versions & Deployments](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/)
  * [Версии и деплои](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/)
- [Rollbacks](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/rollbacks/)
  * [Откаты](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/rollbacks/)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

Частые ошибки (из реальных прогонов)

- **No filesystem in a V8 isolate.** `import fs from "node:fs"` throws at deploy. There is no `/tmp`. If you need to persist anything, it's KV/D1/R2/DO or nothing.
  * **В V8-изоляте нет файловой системы.** `import fs from "node:fs"` падает при деплое. `/tmp` не существует. Если нужно что-то сохранять — это KV/D1/R2/DO или ничего.
- **No Node APIs by default.** `Buffer`, `process`, `crypto` (Node flavor) are not present. To use a subset, add `compatibility_flags = ["nodejs_compat"]` to `wrangler.toml` — this is opt-in for a reason; the V8 isolate is not Node.
  * **По умолчанию нет Node API.** `Buffer`, `process`, `crypto` (вариант Node) отсутствуют. Чтобы использовать подмножество, добавь `compatibility_flags = ["nodejs_compat"]` в `wrangler.toml` — это опция не просто так: V8-изолят не Node.
- **KV is eventually consistent.** A `put` followed immediately by a `get` from a different PoP can return the old value for **up to ~60 seconds**. Fine for counters and flags; wrong for "did the user click the buy button". Use Durable Objects when you need read-your-writes.
  * **KV eventually consistent.** `put`, за которым сразу следует `get` из другого PoP, может вернуть старое значение **до ~60 секунд**. Нормально для счётчиков и флагов; не годится для «пользователь нажал купить». Нужно read-your-writes — бери Durable Objects.
- **`Response.json()` Content-Type.** Auto-set since `compatibility_date: 2022-01-31`. If you pinned an older date (or copy-pasted ancient examples), you have to set `content-type: application/json` by hand. Pin a recent date in `wrangler.toml`.
  * **Content-Type у `Response.json()`.** Выставляется автоматически начиная с `compatibility_date: 2022-01-31`. Если зафиксировал более старую дату (или скопировал древние примеры), придётся вручную ставить `content-type: application/json`. Зафиксируй свежую дату в `wrangler.toml`.
- **`wrangler dev` is local by default in 4.x.** It runs `workerd` on your laptop — `request.cf` is a stub, KV is a SQLite-backed local emulator, secrets come from `.dev.vars`. Use `wrangler dev --remote` only if you need real edge metadata in dev (counts against quota).
  * **`wrangler dev` в 4.x по умолчанию локальный.** Он запускает `workerd` на ноутбуке — `request.cf` там заглушка, KV — локальный эмулятор на SQLite, секреты берутся из `.dev.vars`. Используй `wrangler dev --remote`, только если нужны реальные метаданные периферии в разработке (тратит квоту).
- **`wrangler secret put` requires the Worker to exist on Cloudflare first.** If you haven't run `wrangler deploy` once, the secret command fails. Deploy a stub, then add secrets, then deploy the real version.
  * **`wrangler secret put` требует, чтобы Worker уже существовал в Cloudflare.** Если ты ни разу не выполнял `wrangler deploy`, команда секрета упадёт. Задеплой заглушку, добавь секреты, затем задеплой настоящую версию.
- **`workers.dev` subdomain is one-time-claim per account.** First deploy prompts you to pick `<you>.workers.dev`. You can't change it later without account-level support.
  * **Сабдомен `workers.dev` занимается один раз на аккаунт.** При первом деплое тебя просят выбрать `<you>.workers.dev`. Позже изменить его нельзя без поддержки на уровне аккаунта.
- **`request.cf?.colo` is `undefined` in local dev** (or a constant `"TBS"` stub). Always optional-chain.
  * **`request.cf?.colo` в локальной разработке — `undefined`** (или константная заглушка `"TBS"`). Всегда используй опциональную цепочку.

</details>

<details>
<summary>🐍 Optional: Python Workers</summary>

Опционально: Python Workers

- [Python Workers](https://developers.cloudflare.com/workers/languages/python/) — runs Pyodide; **no C extensions** (no NumPy, no Pillow). If you went Python in Lab 1 and want to keep that path, it's possible but constrained.
  * [Python Workers](https://developers.cloudflare.com/workers/languages/python/) — запускает Pyodide; **без C-расширений** (ни NumPy, ни Pillow). Если ты выбрал Python в лабе 1 и хочешь сохранить этот путь, это возможно, но с ограничениями.
- [Python Worker packages](https://developers.cloudflare.com/workers/languages/python/packages/)
  * [Пакеты для Python Worker](https://developers.cloudflare.com/workers/languages/python/packages/)

</details>

---

## Looking Ahead

Заглядывая вперёд

| Lab | What it adds beyond Lab 17 |
|---:|---|
| 18 | Reproducible builds with Nix — the other bonus / exam-alternative lab. Together: 20% of the grade, full exam replacement |
| — | Workers AI, R2 + R2 Data Catalog, Hyperdrive, Pages Functions — the rest of the Cloudflare developer platform if you want to go deeper after the course |

| Лаба | Что добавляет к лабе 17 |
|---:|---|
| 18 | Воспроизводимые сборки на Nix — вторая бонусная / альтернативная экзамену лаба. Вместе: 20% оценки, полная замена экзамена |
| — | Workers AI, R2 + R2 Data Catalog, Hyperdrive, Pages Functions — остальная часть платформы разработчика Cloudflare, если хочешь копнуть глубже после курса |

**Good luck.** 🌍

**Удачи.** 🌍

> **Remember:** V8 isolates ≠ containers. Workers wins on cold start and price floor for globally distributed APIs; Kubernetes wins on long-running, stateful, native, multi-service workloads. Name the workload first, pick the runtime last.
>
> **Запомни:** V8-изоляты ≠ контейнеры. Workers выигрывает по холодному старту и нижней цене для глобально распределённых API; Kubernetes выигрывает на долгоживущих, stateful, нативных и многосервисных нагрузках. Сначала назови нагрузку, потом выбирай среду выполнения.
