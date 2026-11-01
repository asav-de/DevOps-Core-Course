# DevOps Engineering: Core Practices

DevOps-инженерия: ключевые практики

![main labs](https://img.shields.io/badge/Main_Labs-70%25-blue)
![bonus](https://img.shields.io/badge/Bonuses_+_Bonus_Labs-34%25-yellow)
![exam](https://img.shields.io/badge/Exam-30%25-green)
![duration](https://img.shields.io/badge/Duration-16%20Weeks-lightgrey)
![focus](https://img.shields.io/badge/Focus-Hands--On-orange)

Master **production-grade DevOps** through hands-on labs. You take a single web service and, week by week, containerize it, wire CI/CD, provision its infrastructure, make it observable, orchestrate it on Kubernetes, and ship it through GitOps and progressive delivery — ending with a multi-service, self-healing, fully-observable system and a portfolio that proves you can run software in production.

Осваивай **DevOps продакшн-уровня** на практических лабах. Ты берёшь один веб-сервис и неделя за неделей упаковываешь его в контейнер, подключаешь CI/CD, готовишь инфраструктуру, делаешь его наблюдаемым, оркестрируешь в Kubernetes и доставляешь через GitOps и прогрессивный деплой. В конце — многосервисная, самовосстанавливающаяся, полностью наблюдаемая система и портфолио, которое доказывает, что ты умеешь эксплуатировать софт в продакшене.

> *"You build it, you run it."* — Werner Vogels
>
> *«Ты это построил — ты это и эксплуатируешь»*. — Вернер Фогельс

---

## Course Roadmap

Карта курса

The arc is **build → integrate → provision → observe → orchestrate → deliver → operate at scale**. Each lab builds on the last; your service grows from one container to a three-service cluster.

Логика курса: **создать → интегрировать → подготовить инфраструктуру → наблюдать → оркестрировать → доставить → эксплуатировать в масштабе**. Каждая лаба опирается на предыдущую; твой сервис вырастает из одного контейнера в кластер из трёх сервисов.

| Week | Lab | Module | Key Topics & Tooling (May 2026) |
|:---:|:---:|--------|---------------------------------|
| 1 | 1 | Web Application Development | Python 3.13, Flask/FastAPI, health endpoints |
| 2 | 2 | Containerization | Docker 29, multi-stage, distroless, Trivy scan, GHCR |
| 3 | 3 | Continuous Integration | GitHub Actions, testing pyramid, supply-chain safety |
| 4 | 4 | Infrastructure as Code | Terraform 1.15 **+ Pulumi 3.243**, state, OpenTofu |
| 5 | 5 | Configuration Management | ansible-core 2.21, roles, idempotency, Vault |
| 6 | 6 | Continuous Deployment | Advanced Ansible, Compose v2, CI/CD with OIDC |
| 7 | 7 | Logging | Loki 3.7, **Grafana Alloy 1.16** (Promtail EOL), LogQL |
| 8 | 8 | Monitoring | Prometheus 3.x, PromQL, RED/USE, Grafana 13 |
| 9 | 9 | Kubernetes | K8s 1.36 "Haru", Deployments, Services — **2nd service joins** |
| 10 | 10 | Helm | **Helm 4.1**, charts, templating, hooks, OCI registries |
| 11 | 11 | Secrets Management | K8s Secrets, **OpenBao 2.5**, External Secrets Operator |
| 12 | 12 | Configuration & Storage | ConfigMaps, PV/PVC, StorageClass, hot-reload |
| 13 | 13 | GitOps | **ArgoCD 3.4**, ApplicationSet — **3rd service joins** |
| 14 | 14 | Progressive Delivery | Argo Rollouts 1.8, canary, blue-green, AnalysisTemplate |
| 15 | 15 | StatefulSets | Headless services, volumeClaimTemplates, operators |
| 16 | 16 | Cluster Monitoring | kube-prometheus, ServiceMonitor, init containers |
| — | **Bonus Labs** | | |
| — | 17 | Edge Deployment | Cloudflare Workers, V8 isolates, global edge |
| — | 18 | Reproducible Builds | Nix flakes, deterministic builds |

| Неделя | Лаба | Модуль | Ключевые темы и инструменты (май 2026) |
|:---:|:---:|--------|----------------------------------------|
| 1 | 1 | Разработка веб-приложения | Python 3.13, Flask/FastAPI, эндпоинты health |
| 2 | 2 | Контейнеризация | Docker 29, multi-stage, distroless, сканирование Trivy, GHCR |
| 3 | 3 | Непрерывная интеграция | GitHub Actions, пирамида тестирования, безопасность цепочки поставок |
| 4 | 4 | Инфраструктура как код | Terraform 1.15 **+ Pulumi 3.243**, state, OpenTofu |
| 5 | 5 | Управление конфигурацией | ansible-core 2.21, роли, идемпотентность, Vault |
| 6 | 6 | Непрерывная доставка | Продвинутый Ansible, Compose v2, CI/CD с OIDC |
| 7 | 7 | Логирование | Loki 3.7, **Grafana Alloy 1.16** (Promtail EOL), LogQL |
| 8 | 8 | Мониторинг | Prometheus 3.x, PromQL, RED/USE, Grafana 13 |
| 9 | 9 | Kubernetes | K8s 1.36 «Haru», Deployments, Services — **подключается 2-й сервис** |
| 10 | 10 | Helm | **Helm 4.1**, чарты, шаблонизация, хуки, OCI-реестры |
| 11 | 11 | Управление секретами | K8s Secrets, **OpenBao 2.5**, External Secrets Operator |
| 12 | 12 | Конфигурация и хранилище | ConfigMaps, PV/PVC, StorageClass, hot-reload |
| 13 | 13 | GitOps | **ArgoCD 3.4**, ApplicationSet — **подключается 3-й сервис** |
| 14 | 14 | Прогрессивная доставка | Argo Rollouts 1.8, canary, blue-green, AnalysisTemplate |
| 15 | 15 | StatefulSets | Headless-сервисы, volumeClaimTemplates, операторы |
| 16 | 16 | Мониторинг кластера | kube-prometheus, ServiceMonitor, init-контейнеры |
| — | **Бонусные лабы** | | |
| — | 17 | Доставка на периферию (edge) | Cloudflare Workers, V8 isolates, глобальный edge |
| — | 18 | Воспроизводимые сборки | Nix flakes, детерминированные сборки |

> 📅 **16-week schedule.** If your semester runs shorter, lectures pair up (two per week) so the lab cadence stays one-per-week. Lectures 1-16 map 1:1 to Labs 1-16; bonus labs 17-18 are covered by Lecture 16.
>
> 📅 **Расписание на 16 недель.** Если семестр короче, лекции идут парами (две в неделю), а лабы остаются по одной в неделю. Лекции 1–16 соответствуют лабам 1–16 один к одному; бонусные лабы 17–18 разбираются в лекции 16.

---

## The Project: a service that grows into a system

Проект: сервис, который вырастает в систему

You start with **one** Python service (Lab 1) and never throw it away — every lab adds a production capability to the *same* project. Two course-provided plumbing services join later to make orchestration concepts real:

Ты начинаешь с **одного** Python-сервиса (лаба 1) и больше его не выбрасываешь: каждая лаба добавляет продакшн-возможность в *тот же самый* проект. Позже подключаются два вспомогательных сервиса из курса, чтобы концепции оркестрации стали осязаемыми:

```mermaid
graph LR
    U[User] -->|HTTP| WEB[web :8080<br/>your Python service]
    WEB -->|Lab 9+| ECHO[echo :8081<br/>plumbing]
    WEB -.->|Lab 13+| HEALTH[health :8082<br/>plumbing]

    style WEB fill:#2196F3,color:#fff
    style ECHO fill:#FF9800,color:#fff
    style HEALTH fill:#607D8B,color:#fff
```

Схема: пользователь по HTTP обращается к сервису `web` на порту 8080; с лабы 9 к нему подключается вспомогательный `echo` на 8081, с лабы 13 — `health` на 8082.

| Service | Role | Owner | When it appears |
|---------|------|-------|-----------------|
| **web** | Your Python web service — the project spine | **You build it** | Lab 1 |
| **echo** | Go companion — makes `Service` + kube-DNS meaningful | Course plumbing (`plumbing/echo/`) | Lab 9 |
| **health** | Go companion — gives ArgoCD ApplicationSet a 3rd target | Course plumbing (`plumbing/health/`) | Lab 13 |

| Сервис | Роль | Владелец | Когда появляется |
|--------|------|----------|------------------|
| **web** | Твой Python-веб-сервис — основа проекта | **Строишь ты** | Лаба 1 |
| **echo** | Go-компаньон — делает наглядными `Service` и kube-DNS | Вспомогательный код курса (`plumbing/echo/`) | Лаба 9 |
| **health** | Go-компаньон — даёт ArgoCD ApplicationSet третью цель | Вспомогательный код курса (`plumbing/health/`) | Лаба 13 |

You never write the plumbing services — you deploy and wire them. They expose Prometheus metrics so your Lab 7-8-16 observability stack picks them up automatically.

Ты не пишешь вспомогательные сервисы — ты их разворачиваешь и связываешь с остальными. Они отдают метрики Prometheus, так что стек наблюдаемости из лаб 7, 8 и 16 подхватывает их автоматически.

---

## What Ships vs. What You Produce

Что даёт курс, а что делаешь ты

| In this repo (course-provided) | In YOUR fork (you produce) |
|--------------------------------|----------------------------|
| `lectures/` — 16 lectures | `app_python/` — your service |
| `labs/` — 18 lab specs | `k8s/` — your manifests (Lab 9+) |
| `plumbing/` — echo + health services | `ansible/`, `terraform/` — your IaC (Labs 4-6) |
| `README.md` — this file | `.github/workflows/` — your CI (Lab 3+) |
| | `monitoring/` — your Prometheus/Loki config (Labs 7-8) |

| В этом репозитории (от курса) | В ТВОЁМ форке (делаешь ты) |
|-------------------------------|----------------------------|
| `lectures/` — 16 лекций | `app_python/` — твой сервис |
| `labs/` — 18 спецификаций лаб | `k8s/` — твои манифесты (с лабы 9) |
| `plumbing/` — сервисы echo и health | `ansible/`, `terraform/` — твоя IaC (лабы 4–6) |
| `README.md` — этот файл | `.github/workflows/` — твой CI (с лабы 3) |
| | `monitoring/` — твой конфиг Prometheus/Loki (лабы 7–8) |

Student-produced directories are gitignored in this repo so the upstream stays clean. You commit them to *your* fork.

Каталоги, которые создаёт студент, в этом репозитории находятся в `.gitignore`, чтобы исходный репозиторий оставался чистым. Ты коммитишь их в *свой* форк.

---

## Lab Structure

Структура лаб

Each main lab (1-16) is worth **10 points of main tasks + up to 2 bonus points**:

Каждая основная лаба (1–16) оценивается в **10 баллов за основные задания + до 2 бонусных баллов**:

- **Main tasks** sum to **10 pts**. Split varies by lab (e.g. 6+4, or 2+3+3+2) — Task 1 is always standalone so later labs never depend on a task you skipped.
  - **Основные задания** в сумме дают **10 баллов**. Разбивка зависит от лабы (например, 6+4 или 2+3+3+2); задание 1 всегда самостоятельное, поэтому следующие лабы не зависят от пропущенного задания.
- **Bonus tasks** sum to **2 pts**, flat. Genuinely challenging extensions, not busywork.
  - **Бонусные задания** дают ровно **2 балла**. Это действительно непростые расширения, а не работа ради работы.
- **Bonus labs (17, 18)** follow the same 10 + 2 shape and are the exam-alternative track.
  - **Бонусные лабы (17, 18)** устроены так же (10 + 2) и заменяют экзамен.

Acceptance criteria and a rubric table close every lab. Minimum passing per lab: 6/10.

Каждая лаба заканчивается критериями приёмки и таблицей рубрики. Минимум для зачёта лабы: 6 из 10.

---

## Grading

Оценивание

Five components. Maximum contributions sum to **139%**, capped at **100%** — so there are multiple paths to an A and no single component is mandatory.

Пять компонентов. Максимальная сумма — **139%**, но результат ограничен **100%**, поэтому к «отлично» ведут несколько путей и ни один компонент не обязателен.

| Component | Raw | Weight | Rewards |
|-----------|----:|-------:|---------|
| **Main labs 1-16** (main tasks) | 160 | **70%** | Diligent weekly project work — the floor |
| **Bonus tasks** (2 pts × 16 labs, flat) | 32 | **14%** | Going beyond on weekly topics |
| **Quiz leaderboards** (5 rolling windows, top-10 share a pool) | — | **5%** | Engagement + lecture mastery |
| **Bonus labs 17 + 18** (10 pts each) | 20 | **20%** | Edge + reproducible-build mastery; the exam alternative |
| **Final exam** | — | **30%** | Optional — written, comprehensive |
| **Sum (capped at 100%)** | | **139%** | |

| Компонент | Сырые баллы | Вес | За что |
|-----------|------------:|----:|--------|
| **Основные лабы 1–16** (основные задания) | 160 | **70%** | Регулярная недельная работа над проектом — база |
| **Бонусные задания** (2 балла × 16 лаб) | 32 | **14%** | Выход за рамки недельной темы |
| **Лидерборды квизов** (5 окон, топ-10 делят пул) | — | **5%** | Вовлечённость и знание лекций |
| **Бонусные лабы 17 + 18** (по 10 баллов) | 20 | **20%** | Edge и воспроизводимые сборки; альтернатива экзамену |
| **Финальный экзамен** | — | **30%** | По желанию — письменный, по всему курсу |
| **Сумма (с ограничением 100%)** | | **139%** | |

### Paths to an A (≥90%)

Пути к «отлично» (≥90%)

- **Practice path:** all main labs (70%) + bonuses (14%) + both bonus labs (20%) → 104% → capped A, no exam.
  - **Практический путь:** все основные лабы (70%) + бонусы (14%) + обе бонусные лабы (20%) → 104% → «отлично» без экзамена.
- **Exam path:** all main labs (70%) + bonuses (14%) + a solid exam (30%) → A, no bonus labs.
  - **Экзаменационный путь:** все основные лабы (70%) + бонусы (14%) + хороший экзамен (30%) → «отлично» без бонусных лаб.
- **Mixed:** main labs + some bonuses + one bonus lab + a decent exam.
  - **Смешанный:** основные лабы + часть бонусов + одна бонусная лаба + приличный экзамен.

<details>
<summary>📊 Grade scale</summary>

Шкала оценок:

| Grade | Percentage |
|-------|-----------|
| **A** | 90-100% |
| **B** | 75-89% |
| **C** | 60-74% |
| **D** | below 60% |

| Оценка | Процент |
|--------|---------|
| **A** | 90–100% |
| **B** | 75–89% |
| **C** | 60–74% |
| **D** | ниже 60% |

</details>

---

## Quiz Leaderboards

Лидерборды квизов

Each lecture has a 15-question post-quiz. Quizzes feed **5 rolling leaderboard windows** across the semester:

После каждой лекции идёт квиз из 15 вопросов. Квизы формируют **5 скользящих окон лидерборда** в течение семестра:

| Window | Lectures | Weeks |
|:---:|----------|:---:|
| 1 | lec 1-3 | 1-3 |
| 2 | lec 4-6 | 4-6 |
| 3 | lec 7-9 | 7-9 |
| 4 | lec 10-12 | 10-12 |
| 5 | lec 13-16 | 13-16 |

| Окно | Лекции | Недели |
|:---:|--------|:---:|
| 1 | лекции 1–3 | 1–3 |
| 2 | лекции 4–6 | 4–6 |
| 3 | лекции 7–9 | 7–9 |
| 4 | лекции 10–12 | 10–12 |
| 5 | лекции 13–16 | 13–16 |

The top-10 students in each window split that window's small point pool (≈1% each, ~5% total). Late-joining students can still win later windows.

Топ-10 студентов в каждом окне делят небольшой пул баллов этого окна (примерно по 1% каждому, всего около 5%). Присоединившиеся позже всё ещё могут выиграть в следующих окнах.

---

## Technology Stack (pinned May 2026)

Технологический стек (зафиксирован на май 2026)

| Layer | Tool | Version |
|-------|------|---------|
| Runtime | Python | 3.13 |
| Container | Docker Engine | 29.5 |
| Registry | GHCR | — |
| Scanning | Trivy | v0.69.3+ (post-CVE-2026-33634 safe) |
| CI/CD | GitHub Actions | ubuntu-24.04 runners |
| IaC | Terraform / OpenTofu / Pulumi | 1.15 / 1.12 / 3.243 |
| Config mgmt | ansible-core | 2.21 |
| Logs | Loki + Grafana Alloy | 3.7 / 1.16 |
| Metrics | Prometheus + Grafana | 3.x / 13 |
| Orchestration | Kubernetes | 1.36 "Haru" |
| Local cluster | k3d (k3s-in-Docker) | 5.7 |
| Packaging | Helm | 4.1 |
| Secrets | OpenBao | 2.5 |
| GitOps | ArgoCD | 3.4 |
| Progressive delivery | Argo Rollouts | 1.8 |

| Слой | Инструмент | Версия |
|------|------------|--------|
| Среда исполнения | Python | 3.13 |
| Контейнеры | Docker Engine | 29.5 |
| Реестр образов | GHCR | — |
| Сканирование | Trivy | v0.69.3+ (безопасен после CVE-2026-33634) |
| CI/CD | GitHub Actions | раннеры ubuntu-24.04 |
| IaC | Terraform / OpenTofu / Pulumi | 1.15 / 1.12 / 3.243 |
| Управление конфигурацией | ansible-core | 2.21 |
| Логи | Loki + Grafana Alloy | 3.7 / 1.16 |
| Метрики | Prometheus + Grafana | 3.x / 13 |
| Оркестрация | Kubernetes | 1.36 «Haru» |
| Локальный кластер | k3d (k3s-in-Docker) | 5.7 |
| Упаковка | Helm | 4.1 |
| Секреты | OpenBao | 2.5 |
| GitOps | ArgoCD | 3.4 |
| Прогрессивная доставка | Argo Rollouts | 1.8 |

---

## Submission Workflow

Как сдавать работы

```mermaid
graph LR
    A["Fork repo"] --> B["Branch lab&lt;N&gt;"]
    B --> C["Complete tasks"]
    C --> D["Push & open PR"]
    D --> E["Submit PR URL via Moodle"]
    E --> F["Receive feedback"]
    style A fill:#4CAF50,color:#fff
    style D fill:#F44336,color:#fff
    style E fill:#00BCD4,color:#fff
```

Схема: форк репозитория → ветка `lab<N>` → выполнение заданий → push и открытие PR → отправка URL PR в Moodle → получение обратной связи.

```bash
git checkout -b lab1
# ... complete the lab ...
git add app_python/
git commit -m "Complete lab1"
git push -u origin lab1
# Open a PR from your-fork:lab1 → your-fork:main, submit the PR URL on Moodle
```

Пояснение к командам: создать ветку `lab1`, выполнить лабу, добавить каталог `app_python/`, закоммитить, запушить ветку и открыть PR из `your-fork:lab1` в `your-fork:main`, затем отправить ссылку на PR в Moodle.

<details>
<summary>📝 Submission checklist</summary>

Чек-лист сдачи:

- [ ] All main tasks completed
  - [ ] Все основные задания выполнены
- [ ] Documentation written (`docs/LABNN.md`)
  - [ ] Документация написана (`docs/LABNN.md`)
- [ ] Screenshots/CLI output where required
  - [ ] Скриншоты и вывод CLI там, где требуется
- [ ] Code tested and working
  - [ ] Код протестирован и работает
- [ ] Markdown validated
  - [ ] Markdown проверен
- [ ] PR opened and URL submitted
  - [ ] PR открыт, ссылка отправлена

</details>

---

## Key Books & Resources

Ключевые книги и материалы

**DevOps foundations**
- *The Phoenix Project* — Gene Kim et al. (2013)
- *The DevOps Handbook* (2e) — Kim, Humble, Debois, Willis (2021)
- *Accelerate* — Forsgren, Humble, Kim (2018)

**Основы DevOps**
- *The Phoenix Project* — Джин Ким и др. (2013)
- *The DevOps Handbook* (2-е изд.) — Ким, Хамбл, Дебуа, Уиллис (2021)
- *Accelerate* — Форсгрен, Хамбл, Ким (2018)

**Tooling**
- *Docker Deep Dive* — Nigel Poulton
- *Terraform: Up & Running* (4e) — Yevgeniy Brikman
- *Ansible for DevOps* — Jeff Geerling
- *Kubernetes Up & Running* (3e) — Burns, Beda, Hightower
- *Learning Helm* (2e) — Butcher, Farina, Dolitsky

**Инструменты**
- *Docker Deep Dive* — Найджел Поултон
- *Terraform: Up & Running* (4-е изд.) — Евгений Брикман
- *Ansible for DevOps* — Джефф Гирлинг
- *Kubernetes Up & Running* (3-е изд.) — Бёрнс, Беда, Хайтауэр
- *Learning Helm* (2-е изд.) — Батчер, Фарина, Долицкий

**Observability & reliability**
- *Observability Engineering* — Majors, Fong-Jones, Miranda
- *Site Reliability Engineering* — Beyer et al. (free at sre.google/books) — pairs with the SRE-Intro elective

**Наблюдаемость и надёжность**
- *Observability Engineering* — Мейджорс, Фонг-Джонс, Миранда
- *Site Reliability Engineering* — Байер и др. (бесплатно на sre.google/books) — идёт в паре с элективом SRE-Intro

**Online**
- [DORA State of DevOps](https://dora.dev/research/)
- [CNCF Cloud Native Landscape](https://landscape.cncf.io)
- [12 Factor App](https://12factor.net)

**Онлайн**
- [DORA State of DevOps](https://dora.dev/research/)
- [CNCF Cloud Native Landscape](https://landscape.cncf.io)
- [12 Factor App](https://12factor.net)

---

**Ready? Start with [Lab 1](labs/lab01.md).** Questions → course Moodle or office hours.

**Готов? Начни с [лабы 1](labs/lab01.md).** Вопросы — в Moodle курса или на консультации.
