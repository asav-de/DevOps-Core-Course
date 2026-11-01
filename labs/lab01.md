# Lab 1 — DevOps Info Service: Web Application Development

Лаба 1 — DevOps Info Service: разработка веб-приложения

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Web%20Development-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![languages](https://img.shields.io/badge/languages-Python%20|%20Go-informational)

> **Goal:** Build the seed of your DevOps Info Service — a small HTTP service that reports its own metadata + the host it runs on. Every later lab grows this same service.
> **Deliverable:** A PR from `lab01` with `app_python/` (and optionally `app_go/`). The image you build over the next 16 weeks starts here.
>
> **Цель:** создать основу твоего DevOps Info Service — небольшой HTTP-сервис, который сообщает метаданные о себе и о хосте, на котором работает. Каждая следующая лаба развивает этот же сервис.
> **Результат:** PR из ветки `lab01` с каталогом `app_python/` (и, по желанию, `app_go/`). Образ, который ты построишь за следующие 16 недель, начинается здесь.

---

## Overview

Обзор

In this lab you will practice:

В этой лабе ты попрактикуешься в следующем:

- Picking a Python web framework and justifying the choice
  * Выбрать Python-веб-фреймворк и обосновать выбор
- Reading host/runtime info from `platform`, `socket`, `os`
  * Читать информацию о хосте и среде выполнения из `platform`, `socket`, `os`
- Wiring HTTP routes that return JSON (not HTML)
  * Настроить HTTP-маршруты, возвращающие JSON (а не HTML)
- Pinning dependencies, configuring via environment variables, writing real docs
  * Фиксировать зависимости, настраивать через переменные окружения, писать нормальную документацию

> ⚠️ **Scope:** no database, no auth, no Docker yet. That comes in Lab 2+. Don't get clever — keep the code short and obvious.
>
> ⚠️ **Область работ:** пока без базы данных, без аутентификации и без Docker. Это будет в лабе 2 и дальше. Не мудри — держи код коротким и очевидным.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно остаться от предыдущих лаб:**

- Nothing — this is week 1.
  * Ничего — это первая неделя.

**This lab adds:**

**Эта лаба добавляет:**

- `app_python/` — Python service with `GET /` (info) and `GET /health` (probe)
  * `app_python/` — Python-сервис с `GET /` (информация) и `GET /health` (проверка состояния)
- *(optional bonus)* `app_go/` — compiled-language sibling with the same endpoints
  * *(необязательный бонус)* `app_go/` — аналог на компилируемом языке с теми же эндпоинтами

By Lab 9 this service runs on Kubernetes; by Lab 13 ArgoCD deploys it. So the choices you make this week — framework, layout, env-var config — outlive Lab 1.

К лабе 9 этот сервис будет работать в Kubernetes, а к лабе 13 его будет деплоить ArgoCD. Поэтому решения, принятые на этой неделе — фреймворк, структура, конфигурация через переменные окружения, — переживут лабу 1.

---

## Setup

Подготовка

You need only:

Тебе понадобится только это:

```bash
python3 --version         # 3.12+ (course standardizes on 3.13)
pip3 --version
```

> 📝 **`python` vs `python3`**: on Debian/Ubuntu the binary is `python3`; on macOS/Windows it may be `python`. Pick whichever exists on your box — the rest of this lab writes `python app.py` for brevity, substitute `python3 app.py` if that's how your system spells it.
>
> 📝 **`python` или `python3`:** в Debian/Ubuntu бинарник называется `python3`; в macOS/Windows он может называться `python`. Используй тот, что есть у тебя в системе — дальше в лабе для краткости написано `python app.py`, замени на `python3 app.py`, если у тебя принято так.

Create the directory layout (you'll fill the files yourself):

Создай структуру каталогов (файлы ты заполнишь сам):

```
app_python/
├── app.py
├── requirements.txt
├── .gitignore
├── README.md
├── tests/
│   └── __init__.py            # empty; tests come in Lab 3
└── docs/
    └── LAB01.md               # your submission report
```

Пояснение: создаётся каталог `app_python/` со скриптом приложения, файлом зависимостей, `.gitignore`, README, пустым пакетом `tests/` и каталогом `docs/` для отчёта.

---

## Task 1 — Python Web Application (6 pts)

Задача 1 — Python-веб-приложение (6 баллов)

### 1.1 — Pick a framework and justify it

1.1 — Выбери фреймворк и обоснуй выбор

You may use **Flask 3.1**, **FastAPI 0.136**, or **Django 5.2** (overkill for this; do it if you want the practice). In `docs/LAB01.md`, give your one-paragraph reasoning + a small comparison table covering at least: footprint, async support, OpenAPI/docs, learning curve.

Можешь использовать **Flask 3.1**, **FastAPI 0.136** или **Django 5.2** (для этой задачи это избыточно; бери, если хочешь попрактиковаться). В `docs/LAB01.md` приведи обоснование на один абзац и небольшую сравнительную таблицу как минимум по таким параметрам: занимаемое место, поддержка асинхронности, OpenAPI/документация, кривая обучения.

### 1.2 — `GET /` returns the full info JSON

1.2 — `GET /` возвращает полный JSON с информацией

`YOUR TASK`: write a handler at the root path that returns JSON with **five** top-level keys: `service`, `system`, `runtime`, `request`, `endpoints`.

`YOUR TASK`: напиши обработчик корневого пути, который возвращает JSON с **пятью** ключами верхнего уровня: `service`, `system`, `runtime`, `request`, `endpoints`.

Requirements:

Требования:

- `service`: `name`, `version`, `description`, `framework` (your choice from 1.1)
  * `service`: `name`, `version`, `description`, `framework` (твой выбор из 1.1)
- `system`: `hostname`, `platform`, `platform_version`, `architecture`, `cpu_count`, `python_version`
  * `system`: поля `hostname`, `platform`, `platform_version`, `architecture`, `cpu_count`, `python_version`
- `runtime`: `uptime_seconds` (int), `uptime_human` (e.g. `"1 hour, 3 minutes"`), `current_time` (ISO 8601 UTC), `timezone`
  * `runtime`: `uptime_seconds` (int), `uptime_human` (например, `"1 hour, 3 minutes"`), `current_time` (ISO 8601 UTC), `timezone`
- `request`: `client_ip`, `user_agent`, `method`, `path`
  * `request`: данные о клиенте — `client_ip`, `user_agent`, `method`, `path`
- `endpoints`: list of `{path, method, description}` for every route you publish
  * `endpoints`: список `{path, method, description}` для каждого публикуемого маршрута

Hints:

Подсказки:

- `platform.system()`, `platform.machine()`, `platform.python_version()` cover most of `system`
  * `platform.system()`, `platform.machine()`, `platform.python_version()` покрывают большую часть `system`
- `socket.gethostname()` for the hostname
  * `socket.gethostname()` — для имени хоста
- Uptime = `datetime.now(timezone.utc) - START_TIME` captured at module import — be **timezone-aware** throughout (mixing naive + aware datetimes raises `TypeError`)
  * Uptime = `datetime.now(timezone.utc) - START_TIME`, зафиксированное при импорте модуля. Везде работай с **timezone-aware** датами: смешивание naive- и aware-дат приводит к `TypeError`
- Per-framework request access: Flask → `request.remote_addr` / `request.headers.get('User-Agent')`; FastAPI → `request.client.host` / `request.headers.get('user-agent')`
  * Доступ к запросу во фреймворках: Flask → `request.remote_addr` / `request.headers.get('User-Agent')`; FastAPI → `request.client.host` / `request.headers.get('user-agent')`

<details>
<summary>💡 Stuck on the structure?</summary>

💡 Застрял на структуре?

Sketch — no Python code given, intentionally:

Набросок — Python-код намеренно не приводится:

```
imports → app object → constants (SERVICE_NAME, START_TIME, ...) → helper get_system_info() → helper get_uptime() → route handler returning the 5-key dict → error handlers → __main__ block
```

Пояснение: порядок такой — импорты, объект приложения, константы (`SERVICE_NAME`, `START_TIME`, ...), вспомогательная функция `get_system_info()`, функция `get_uptime()`, обработчик маршрута, возвращающий словарь с пятью ключами, обработчики ошибок и блок `__main__`.

Lecture 1 slide on the lifecycle, lecture 2 on configuration — both apply here.

Слайд лекции 1 о жизненном цикле и лекция 2 о конфигурации — обе применимы здесь.

</details>

### 1.3 — `GET /health` returns a small JSON + HTTP 200

1.3 — `GET /health` возвращает небольшой JSON и HTTP 200

`YOUR TASK`: write a second handler that returns `{status, timestamp, uptime_seconds}` and status code **200**. This will be your Kubernetes liveness/readiness target in Lab 9 — keep it cheap (no DB calls, no external HTTP).

`YOUR TASK`: напиши второй обработчик, который возвращает `{status, timestamp, uptime_seconds}` и код **200**. В лабе 9 это будет цель для Kubernetes liveness/readiness-проверок — держи его дешёвым (без запросов к БД и внешних HTTP-вызовов).

### 1.4 — Configurable via environment variables

1.4 — Конфигурация через переменные окружения

`YOUR TASK`: read `HOST`, `PORT`, `DEBUG` from the environment with sensible defaults (`0.0.0.0`, `5000`, `False`). The app must run unchanged with:

`YOUR TASK`: читай `HOST`, `PORT`, `DEBUG` из окружения с разумными значениями по умолчанию (`0.0.0.0`, `5000`, `False`). Приложение должно работать без изменений при таком запуске:

```bash
python app.py                            # default 0.0.0.0:5000
PORT=8080 python app.py                  # custom port
HOST=127.0.0.1 PORT=3000 python app.py   # both
```

Hint: cast `PORT` to `int` and `DEBUG` to `bool` by **lowercase string comparison** (`os.getenv("DEBUG","False").lower() == "true"`) — `bool("False")` is `True` in Python; that's the easy way to ship a debug-mode bug to production.

Подсказка: приводи `PORT` к `int`, а `DEBUG` к `bool` через **сравнение строк в нижнем регистре** (`os.getenv("DEBUG","False").lower() == "true"`) — в Python `bool("False")` равно `True`; именно так легко утащить баг с debug-режимом в прод.

### 1.5 — Error handlers and logging

1.5 — Обработчики ошибок и логирование

`YOUR TASK`: register handlers for **404** and **500** that return JSON (matching the rest of the API), and configure stdlib `logging` at INFO level with a format including timestamp/level. Don't `print()`.

`YOUR TASK`: зарегистрируй обработчики для **404** и **500**, возвращающие JSON (как и остальное API), и настрой стандартный `logging` на уровне INFO с форматом, включающим время и уровень. Не используй `print()`.

### 1.6 — Proof of work

1.6 — Подтверждение работы

**Paste into `docs/LAB01.md`:**

**Вставь в `docs/LAB01.md`:**

- Four CLI captures, with the **exact commands** you ran:
  * Четыре снимка вывода CLI с **точными командами**, которые ты запускал:
  - `curl -s http://localhost:5000/ | jq .` — full JSON, your real hostname + uptime
    * `curl -s http://localhost:5000/ | jq .` — полный JSON, твой реальный hostname и uptime
  - `curl -s http://localhost:5000/health | jq .`
    * `curl -s http://localhost:5000/health | jq .` — JSON-ответ проверки состояния
  - `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:5000/nope` — proves the 404 handler
    * `curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:5000/nope` — доказывает работу обработчика 404
  - `PORT=8080 python app.py` started in another shell + a `curl` showing the new port served — proves env-var config
    * `PORT=8080 python app.py`, запущенная в другом терминале, плюс `curl`, показывающий, что новый порт обслуживается — доказывает работу конфигурации через переменные окружения
- The framework comparison table from 1.1
  * Сравнительная таблица фреймворков из 1.1
- 2–3 sentences on the biggest gotcha you hit and how you solved it
  * 2–3 предложения о самой большой подводной камне, на которую ты наткнулся, и о том, как ты её решил

<details>
<summary>💡 Hints if you're stuck</summary>

💡 Подсказки, если застрял

- Make the framework choice early — switching mid-lab burns time.
  * Выбирай фреймворк сразу — переключение посреди лабы съедает время.
- Build `GET /` last. Build `GET /health` first (5 lines), then incrementally add system info, uptime, request info.
  * Собери `GET /` последним. Сначала сделай `GET /health` (5 строк), затем постепенно добавляй информацию о системе, uptime и данные запроса.
- If JSON output looks "weird" (e.g., bytes prefix `b'...'`), you're returning a string instead of a dict — let the framework jsonify.
  * Если JSON выглядит «странно» (например, с префиксом байтов `b'...'`), значит, ты возвращаешь строку, а не словарь — дай фреймворку сериализовать её в JSON.

</details>

---

## Task 2 — Documentation & Best Practices (4 pts)

Задача 2 — Документация и лучшие практики (4 балла)

### 2.1 — `app_python/README.md`

2.1 — Файл `app_python/README.md`

`YOUR TASK`: write the user-facing README with these sections (no fluff, prefer commands over prose):

`YOUR TASK`: напиши README для пользователя со следующими разделами (без воды, команды предпочтительнее прозы):

1. **Overview** — one paragraph
   * **Обзор** — один абзац
2. **Prerequisites** — Python version, OS
   * **Требования** — версия Python, ОС
3. **Installation** — `python -m venv venv` … `pip install -r requirements.txt`
   * **Установка** — `python -m venv venv` … `pip install -r requirements.txt`
4. **Running** — default + custom-port examples
   * **Запуск** — примеры с портом по умолчанию и с другим портом
5. **API Endpoints** — table of `Method | Path | Description`
   * **Эндпоинты API** — таблица `Method | Path | Description`
6. **Configuration** — table of env vars + defaults + purpose
   * **Конфигурация** — таблица переменных окружения + значения по умолчанию + назначение

### 2.2 — Code-quality hygiene

2.2 — Гигиена качества кода

`YOUR TASK`: ship the things every Python service needs.

`YOUR TASK`: подготовь то, что нужно каждому Python-сервису.

| File | Must contain |
|---|---|
| `requirements.txt` | The framework **pinned to an exact patch** (e.g. `Flask==3.1.3`, not `Flask>=3`). One framework only — drop the rest. |
| `.gitignore` | `__pycache__/`, `*.pyc`, `venv/`, `*.log`, IDE dirs (`.vscode/`, `.idea/`), `.DS_Store` |
| Imports in `app.py` | Grouped (stdlib → third-party → local), PEP 8 compliant |

| Файл | Что должно быть |
|---|---|
| `requirements.txt` | Фреймворк, **зафиксированный на точной версии патча** (например, `Flask==3.1.3`, а не `Flask>=3`). Только один фреймворк — остальное убери. |
| `.gitignore` | `__pycache__/`, `*.pyc`, `venv/`, `*.log`, каталоги IDE (`.vscode/`, `.idea/`), `.DS_Store` |
| Импорты в `app.py` | Сгруппированы (stdlib → сторонние → локальные), соответствуют PEP 8 |

### 2.3 — `app_python/docs/LAB01.md` — your submission report

2.3 — `app_python/docs/LAB01.md` — отчёт о выполнении

Required sections, in order:

Обязательные разделы по порядку:

1. **Framework Selection** — your choice + comparison table
   * **Выбор фреймворка** — твой выбор + сравнительная таблица
2. **Best Practices Applied** — bulleted list, each with one sentence of *why*
   * **Применённые лучшие практики** — список, где у каждого пункта одно предложение о том, *почему*
3. **API Documentation** — request + response example per endpoint
   * **Документация API** — пример запроса и ответа для каждого эндпоинта
4. **Testing Evidence** — the four CLI captures from 1.6
   * **Доказательства тестирования** — четыре снимка вывода CLI из 1.6
5. **Challenges & Solutions** — at least one real one (not "I was new to Python")
   * **Трудности и решения** — хотя бы одна настоящая (не «я был новичком в Python»)
6. **GitHub Community** — one paragraph; see 2.4
   * **GitHub-сообщество** — один абзац; см. 2.4

### 2.4 — GitHub community engagement

2.4 — Участие в GitHub-сообществе

This is the social half of being a developer. The actions are public on your profile.

Это социальная половина работы разработчика. Эти действия видны публично в твоём профиле.

`YOUR TASK`:

`YOUR TASK`:

1. **Star** the course repository.
   * **Поставь звезду** репозиторию курса.
2. **Star** the [simple-container-com/api](https://github.com/simple-container-com/api) project — a promising open-source tool for container management.
   * **Поставь звезду** проекту [simple-container-com/api](https://github.com/simple-container-com/api) — перспективному open-source-инструменту для управления контейнерами.
3. **Follow** your professor and TAs on GitHub:
   * **Подпишись** на преподавателя и TA в GitHub:
   - Professor: [@Cre-eD](https://github.com/Cre-eD)
      * Преподаватель: [@Cre-eD](https://github.com/Cre-eD)
   - TA: [@Naghme98](https://github.com/Naghme98)
      * TA: [@Naghme98](https://github.com/Naghme98)
   - TA: [@pierrepicaud](https://github.com/pierrepicaud)
      * TA: [@pierrepicaud](https://github.com/pierrepicaud)
4. **Follow** ≥ 3 classmates from this course.
   * **Подпишись** минимум на 3 однокурсников с этого курса.
5. In your `docs/LAB01.md` "GitHub Community" section, **put your GitHub username on the first line** (so the TA can verify your stars/follows in <30s) then write 2–3 sentences answering: *Why does starring a project actually help its maintainers? What concrete benefit do you get from following other developers?* — your own words, not the lecture's.
   * В разделе «GitHub Community» файла `docs/LAB01.md` **укажи свой GitHub-username первой строкой** (чтобы TA мог проверить твои звёзды и подписки меньше чем за 30 секунд), затем напиши 2–3 предложения с ответом: *Почему звезда проекту на самом деле помогает его мейнтейнерам? Какую конкретную пользу ты получаешь, подписываясь на других разработчиков?* — своими словами, не из лекции.

### 2.5 — Proof of work

2.5 — Подтверждение работы

**Paste into `docs/LAB01.md`:**

**Вставь в `docs/LAB01.md`:**

- The `docs/LAB01.md` itself satisfies all six required sections — that *is* the proof.
  * Сам `docs/LAB01.md` удовлетворяет всем шести обязательным разделам — *это и есть* доказательство.
- Output of `find app_python -maxdepth 2 -type f | sort` showing every required file is present.
  * Вывод `find app_python -maxdepth 2 -type f | sort`, показывающий, что все обязательные файлы на месте.

---

## Bonus Task — Compiled-Language Sibling (2 pts)

Бонусная задача — аналог на компилируемом языке (2 балла)

Re-implement the same service in a **compiled language**, in a sibling directory (e.g. `app_go/`).

Реализуй тот же сервис на **компилируемом языке** в соседнем каталоге (например, `app_go/`).

Why bother? Lab 2's multi-stage Docker bonus shrinks a Go service from ~900 MB to ~15 MB — that doesn't work without a static binary to start with.

Зачем? Бонус с multi-stage Docker во второй лабе уменьшит Go-сервис с ~900 МБ до ~15 МБ — но это не сработает без статического бинарника на входе.

`YOUR TASK`:

`YOUR TASK`:

- Same two endpoints (`/`, `/health`)
  * Те же два эндпоинта (`/`, `/health`)
- Same JSON shape as your Python version (so the same `curl` calls work against both)
  * Та же форма JSON, что в Python-версии (чтобы одни и те же вызовы `curl` работали с обеими)
- A README in the sibling dir explaining build + run
  * README в соседнем каталоге с описанием сборки и запуска
- A `go.mod` (and `go.sum` if you pull deps) — Lab 2's bonus needs them as the build context
  * `go.mod` (и `go.sum`, если тянешь зависимости) — они нужны бонусу из лабы 2 как build context
- In `docs/LAB01.md`, add a one-line **artifact size** for both. Measure the same way for each so the comparison is apples-to-apples:
  * В `docs/LAB01.md` добавь по одной строке с **размером артефакта** для обеих версий. Измеряй одинаково, чтобы сравнение было честным:
  - Python: `du -sh app_python/venv` (the venv carries everything beyond the interpreter)
    * Python: `du -sh app_python/venv` (venv содержит всё, что сверх интерпретатора)
  - Compiled: `ls -lh app_go/<binary>` (the single static binary)
    * Компилируемая версия: `ls -lh app_go/<binary>` (единственный статический бинарник)
  - The size delta is the whole point. The real Docker-image comparison comes in Lab 2.
    * Вся суть — в разнице размеров. Настоящее сравнение Docker-образов будет в лабе 2.

Choose one of:

Выбери один из вариантов:

| Language | Idiomatic HTTP | Notes |
|---|---|---|
| **Go** *(recommended)* | `net/http` | Single static binary; instant compile; smallest distroless image later |
| Rust | `actix-web` / `axum` | Strong types; longer compile; great for the security-minded |
| Java + Spring Boot | `spring-boot-starter-web` | Industry-standard for enterprise — heavier runtime |
| C# + ASP.NET Core | `WebApplication.Create` | Cross-platform .NET; comparable to Java but newer ergonomics |

| Язык | Идиоматичный HTTP | Замечания |
|---|---|---|
| **Go** *(рекомендуется)* | `net/http` | Один статический бинарник; мгновенная компиляция; позже — самый маленький distroless-образ |
| Rust | `actix-web` / `axum` | Сильная типизация; дольше компиляция; отличный выбор для тех, кто думает о безопасности |
| Java + Spring Boot | `spring-boot-starter-web` | Отраслевой стандарт для энтерпрайза — тяжелее runtime |
| C# + ASP.NET Core | `WebApplication.Create` | Кроссплатформенный .NET; сопоставим с Java, но с более современным API |

Hints (no full code):

Подсказки (без полного кода):

- For Go, `runtime.NumCPU()`, `runtime.GOOS`, `runtime.GOARCH`, `os.Hostname()` mirror Python's `os`/`platform`. JSON keys differ in casing — use struct tags (`json:"hostname"`) to keep the same wire shape.
  * В Go `runtime.NumCPU()`, `runtime.GOOS`, `runtime.GOARCH`, `os.Hostname()` повторяют `os`/`platform` из Python. Регистр JSON-ключей отличается — используй теги структур (`json:"hostname"`), чтобы сохранить одинаковую форму по сети.
- For Java/C#, start from `spring init` / `dotnet new web` — don't hand-write `pom.xml`/`csproj`.
  * В Java/C# начинай с `spring init` / `dotnet new web` — не пиши `pom.xml`/`csproj` вручную.

---

## How to Submit

Как сдавать

```bash
git switch -c lab01
git add app_python/
git add app_go/                   # only if you did the bonus
git commit -m "feat(lab01): devops info service — python (+ go bonus)"
git push -u origin lab01
```

Пояснение: создай ветку `lab01`, добавь каталог `app_python/` (и `app_go/`, если делал бонус), закоммить и запушь ветку в свой форк.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab01` → `course-repo:master` *(reviewed)*
  * `your-fork:lab01` → `course-repo:master` *(с ревью)*
- `your-fork:lab01` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab01` → `your-fork:master` *(сливается в твой собственный main, когда закончишь)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — /, /health, env config, error handlers, logging
- [ ] Task 2 done — README, requirements.txt, .gitignore, docs/LAB01.md, GitHub social
- [ ] Bonus done — app_go/ (or other) with same endpoints + size comparison
```

Пояснение: в чек-листе отмечаются задача 1 (`/`, `/health`, конфигурация через env, обработчики ошибок, логирование), задача 2 (README, `requirements.txt`, `.gitignore`, `docs/LAB01.md`, GitHub-активность) и бонус (`app_go/` или другой язык с теми же эндпоинтами и сравнением размеров).

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (6 pts)

Задача 1 (6 баллов)

- ✅ Service starts with `python app.py` and serves on `0.0.0.0:5000`
  * ✅ Сервис запускается командой `python app.py` и обслуживает `0.0.0.0:5000`
- ✅ `GET /` returns HTTP 200 with all five top-level keys populated from *your real machine*
  * ✅ `GET /` возвращает HTTP 200, и все пять ключей верхнего уровня заполнены данными с *твоей реальной машины*
- ✅ `GET /health` returns HTTP 200 with the three required fields
  * ✅ `GET /health` возвращает HTTP 200 с тремя обязательными полями
- ✅ 404 handler returns JSON (not HTML)
  * ✅ Обработчик 404 возвращает JSON (а не HTML)
- ✅ `PORT` env var overrides the port
  * ✅ Переменная окружения `PORT` переопределяет порт
- ✅ Logging is configured (you'll see startup log line in stdout)
  * ✅ Логирование настроено (в stdout видна строка лога при запуске)

### Task 2 (4 pts)

Задача 2 (4 балла)

- ✅ `README.md` has all six sections
  * ✅ В `README.md` есть все шесть разделов
- ✅ `requirements.txt` pins an **exact** version
  * ✅ В `requirements.txt` зафиксирована **точная** версия
- ✅ `.gitignore` covers Python + IDE + OS artifacts
  * ✅ `.gitignore` покрывает артефакты Python, IDE и ОС
- ✅ `docs/LAB01.md` has all six sections including the GitHub Community paragraph
  * ✅ В `docs/LAB01.md` есть все шесть разделов, включая абзац о GitHub-сообществе
- ✅ Stars + follows visible on student's GitHub profile
  * ✅ Звёзды и подписки видны в GitHub-профиле студента

### Bonus Task (2 pts)

Бонусная задача (2 балла)

- ✅ Sibling app builds and serves the same two endpoints
  * ✅ Соседнее приложение собирается и обслуживает те же два эндпоинта
- ✅ Wire-compatible JSON (same `curl … | jq` works against both)
  * ✅ JSON совместим по сети (одни и те же `curl … | jq` работают с обеими версиями)
- ✅ Artifact-size comparison documented (`du -sh venv` vs `ls -lh <binary>`)
  * ✅ Сравнение размеров артефактов задокументировано (`du -sh venv` против `ls -lh <binary>`)
- ✅ Build manifest present (`go.mod` / `Cargo.toml` / `pom.xml` / `*.csproj`) so Lab 2's bonus has a build context to `COPY`
  * ✅ Присутствует манифест сборки (`go.mod` / `Cargo.toml` / `pom.xml` / `*.csproj`), чтобы бонусу из лабы 2 было что `COPY` в build context

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Python web app | **6** | Both routes correct, env-var config, error handlers, logging |
| **Task 2** — Docs & hygiene | **4** | All file/section requirements met; pinned deps; complete LAB01.md |
| **Bonus** — Compiled sibling | **2** | Same endpoints, wire-compatible JSON, size comparison |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — Python-веб-приложение | **6** | Оба маршрута корректны, конфигурация через env, обработчики ошибок, логирование |
| **Задача 2** — Документация и гигиена | **4** | Выполнены все требования к файлам и разделам; зависимости зафиксированы; LAB01.md заполнен полностью |
| **Бонус** — Аналог на компилируемом языке | **2** | Те же эндпоинты, совместимый JSON, сравнение размеров |
| **Итого** | **12** | 10 основных + 2 бонусных |

---

## Resources

Ресурсы

<details>
<summary>📚 Documentation</summary>

📚 Документация

- [Flask 3.1](https://flask.palletsprojects.com/en/latest/) — the canonical micro-framework
  * [Flask 3.1](https://flask.palletsprojects.com/en/latest/) — канонический микрофреймворк
- [FastAPI](https://fastapi.tiangolo.com/) — async, OpenAPI free
  * [FastAPI](https://fastapi.tiangolo.com/) — асинхронность, OpenAPI из коробки
- [Django 5.2](https://docs.djangoproject.com/en/stable/) — full-stack; overkill for this lab
  * [Django 5.2](https://docs.djangoproject.com/en/stable/) — full-stack; для этой лабы избыточен
- [Python `platform`](https://docs.python.org/3/library/platform.html) / [`socket`](https://docs.python.org/3/library/socket.html)
  * Документация Python: [`platform`](https://docs.python.org/3/library/platform.html) / [`socket`](https://docs.python.org/3/library/socket.html)
- [PEP 8](https://pep8.org/), [PEP 660](https://peps.python.org/pep-0660/)
  * [PEP 8](https://pep8.org/), [PEP 660](https://peps.python.org/pep-0660/) — стиль кода и метаданные пакетов

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **Naive vs aware datetimes** — `datetime.now() - START_TIME` raises `TypeError` if one is timezone-aware and the other isn't. Pick one (use `datetime.now(timezone.utc)` everywhere) and stay there.
  * **Naive- и aware-даты** — `datetime.now() - START_TIME` вызывают `TypeError`, если одна дата timezone-aware, а другая нет. Выбери один вариант (везде используй `datetime.now(timezone.utc)`) и придерживайся его.
- **`bool("False") == True`** — never write `DEBUG = bool(os.getenv("DEBUG", "False"))`. Use a lowercase string compare.
  * **`bool("False") == True`** — никогда не пиши `DEBUG = bool(os.getenv("DEBUG", "False"))`. Используй сравнение строк в нижнем регистре.
- **Flask 3 deprecation warning** on `flask.__version__` — use `importlib.metadata.version("flask")` if you need to print the framework version.
  * **Предупреждение об устаревании во Flask 3** при обращении к `flask.__version__` — если нужно вывести версию фреймворка, используй `importlib.metadata.version("flask")`.
- **Returning a dict directly from FastAPI vs Flask** — FastAPI auto-serializes; Flask needs `jsonify(...)`. Don't return a bare dict from a Flask handler.
  * **Возврат словаря напрямую: FastAPI против Flask** — FastAPI сериализует автоматически; Flask требует `jsonify(...)`. Не возвращай «голый» словарь из обработчика Flask.
- **`socket.gethostname()` inside a container** later returns the container ID, not your laptop name — that's correct, not a bug. You'll see it again in Lab 2.
  * **`socket.gethostname()` внутри контейнера** позже вернёт ID контейнера, а не имя твоего ноутбука — это правильно, а не баг. Ты увидишь это снова в лабе 2.
- **Port 5000 occupied on macOS** — macOS Monterey+ runs the *AirPlay Receiver* on `:5000`. The Flask default conflicts; you'll see your `curl` hit Apple's service instead. Either turn AirPlay Receiver off (System Settings → General → AirDrop & Handoff) or pick `PORT=5050` for your default run.
  * **Порт 5000 занят на macOS** — macOS Monterey и новее запускает *AirPlay Receiver* на `:5000`. Конфликт с портом Flask по умолчанию: твой `curl` попадёт в сервис Apple. Либо отключи AirPlay Receiver (System Settings → General → AirDrop & Handoff), либо задай `PORT=5050` для запуска по умолчанию.
- **`pip install -r requirements.txt` outside the venv** — installs Flask system-wide, then `python app.py` may still pick up a stale Flask elsewhere. Always activate the venv (`source venv/bin/activate`) before `pip install` and before `python app.py`.
  * **`pip install -r requirements.txt` вне venv** — Flask установится в систему, и потом `python app.py` может подхватить устаревший Flask из другого места. Всегда активируй venv (`source venv/bin/activate`) перед `pip install` и перед `python app.py`.
- **Don't return a Response from `@app.errorhandler`'s default** — Flask's 500 handler must return a tuple `(body, status)`. Returning a bare `jsonify(...)` sends HTTP 200 with the error JSON — a subtle bug that breaks the Lab 9 probe later.
  * **Не возвращай Response из обработчика `@app.errorhandler` по умолчанию** — обработчик 500 во Flask должен вернуть кортеж `(body, status)`. Если вернуть просто `jsonify(...)`, ответ уйдёт с HTTP 200 и JSON ошибки — незаметный баг, который позже сломает проверку в лабе 9.

</details>

<details>
<summary>🛠️ Dev tools worth knowing</summary>

🛠️ Полезные инструменты разработчика

- [jq](https://jqlang.github.io/jq/) — JSON CLI; `curl ... | jq .` is your friend
  * [jq](https://jqlang.github.io/jq/) — CLI для JSON; `curl ... | jq .` — твой друг
- [HTTPie](https://httpie.io/) — friendlier than `curl` for ad-hoc testing
  * [HTTPie](https://httpie.io/) — удобнее `curl` для быстрых проверок
- [Ruff](https://docs.astral.sh/ruff/) — fast Python linter (used in Lab 3 CI)
  * [Ruff](https://docs.astral.sh/ruff/) — быстрый линтер Python (используется в CI лабы 3)

</details>

---

## Looking Ahead

Что впереди

| Lab | What it adds to this service |
|---:|---|
| 2 | Multi-stage Dockerfile, image scan, push to a registry |
| 3 | CI: pytest + lint + image build + Trivy gate on every PR |
| 7 / 8 | Structured JSON logs (Loki + Alloy) and a `/metrics` endpoint (Prometheus) |
| 9 / 10 | Deploy to Kubernetes (k3d) + Helm 4 chart |
| 11 / 12 | OpenBao for secrets, ConfigMaps + PVCs for state |
| 13 / 14 | ArgoCD GitOps + canary rollouts |

| Лаба | Что добавляет к сервису |
|---:|---|
| 2 | Многоступенчатый Dockerfile, сканирование образа, пуш в registry |
| 3 | CI: pytest + линтер + сборка образа + проверка Trivy на каждом PR |
| 7 / 8 | Структурированные JSON-логи (Loki + Alloy) и эндпоинт `/metrics` (Prometheus) |
| 9 / 10 | Деплой в Kubernetes (k3d) + Helm 4 chart |
| 11 / 12 | OpenBao для секретов, ConfigMaps + PVC для состояния |
| 13 / 14 | GitOps на ArgoCD + canary-раскатки |

Keep the code simple. You'll come back and rewrite parts of it; that's the point.

Держи код простым. Ты вернёшься и перепишешь часть — в этом и смысл.
