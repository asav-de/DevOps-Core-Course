# Lab 14 — Progressive Delivery with Argo Rollouts

Лаба 14 — прогрессивная доставка с Argo Rollouts

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Progressive%20Delivery-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Argo%20Rollouts%201.8.4-informational)

> **Goal:** Replace `kind: Deployment` with `kind: Rollout`. Write the canary `steps:` list yourself, write the active + preview `Service` manifests yourself, and (bonus) write the PromQL behind a metric gate that auto-aborts a regressing release.
>
> **Цель:** заменить `kind: Deployment` на `kind: Rollout`. Сам напиши список `steps:` канареечного rollout, сам напиши манифесты `Service` для active и preview, а бонусом — PromQL за метрическим гейтом, который автоматически откатывает регрессирующий релиз.
>
> **Deliverable:** A PR from `lab14` with `k8s/lab10-app/templates/rollout.yaml`, the two blue-green `Service`s, optional `AnalysisTemplate`, and `docs/LAB14.md`.
>
> **Что сдать:** PR из ветки `lab14` с файлом `k8s/lab10-app/templates/rollout.yaml`, двумя сине-зелёными `Service`, необязательным `AnalysisTemplate` и `docs/LAB14.md`.

---

## Overview

Обзор

Through Lab 9 you ran rolling updates with a vanilla `Deployment`: K8s swaps pods 25% at a time and stops measuring the moment a readiness probe flips green. The 5xx spike that only fires on 5% of requests still ships to 100% of users. **Progressive delivery** closes that loop — shape traffic in steps, gate each step behind metrics, and let the controller roll back without a human watching Grafana on a Friday.

С лабы 9 ты делал rolling update обычным `Deployment`: Kubernetes заменяет поды по 25% за раз и перестаёт что-либо измерять, как только readiness-проба становится зелёной. Всплеск 5xx, который возникает лишь на 5% запросов, всё равно доезжает до 100% пользователей. **Прогрессивная доставка** замыкает этот контур: управляй трафиком по шагам, ставь за каждым шагом метрический гейт и позволь контроллеру откатиться сам, без человека, следящего за Grafana в пятницу вечером.

In this lab you will practice:

В этой лабе ты отработаешь:

- The `Rollout` CRD as a drop-in for `Deployment` — same pod template, new `spec.strategy:` block
  * CRD `Rollout` как полноценная замена `Deployment` — тот же pod-шаблон, новый блок `spec.strategy:`
- **Writing the canary `steps:` list from scratch** — every `setWeight` and every `pause` is yours
  * **Написание списка `steps:` для канареечного rollout с нуля** — каждый `setWeight` и каждая `pause` — твоя работа
- **Writing the active and preview `Service` manifests from scratch** for blue-green
  * **Написание манифестов `Service` для active и preview с нуля** для сине-зелёного деплоя
- **Writing the Prometheus query and gate conditions** in an `AnalysisTemplate` for the bonus
  * **Написание запроса Prometheus и условий гейта** в `AnalysisTemplate` для бонуса
- Driving rollouts with `kubectl argo rollouts <verb>` — and learning the verbs by name
  * Управление rollout'ами через `kubectl argo rollouts <verb>` — и запоминание названий команд

> ⚠️ **Scope:** No traffic-router integration in the main path — `setWeight` approximates by replica ratio (lecture 14 slide 12). One traffic-router option lives in the bonus. No mesh, no notifications controller. Stick to the canary + blue-green core and write every line of strategy yourself.
>
> ⚠️ **Объём:** в основной части нет интеграции с трафик-роутером — `setWeight` приближает распределение через соотношение реплик (лекция 14, слайд 12). Один вариант с трафик-роутером вынесен в бонус. Ни service mesh, ни контроллера уведомлений. Держись ядра canary + blue-green и сам напиши каждую строку стратегии.

> 🪨 **Pedagogical core.** The headline proof of this lab is a single screen: `kubectl argo rollouts get rollout lab10-app-web` showing **`Status: ॥ Paused`**, **`Message: CanaryPauseStep`**, and **`SetWeight: 25, ActualWeight: 25`** at step 1/6 — then, after you run `promote --full`, the same command at **step 6/6 `Healthy`**. A `Rollout` that goes straight to 100% with no visible pause didn't exercise progressive delivery. **Save that capture.**
>
> 🪨 **Педагогическое ядро.** Главное доказательство этой лабы — один экран: `kubectl argo rollouts get rollout lab10-app-web` показывает **`Status: ॥ Paused`**, **`Message: CanaryPauseStep`** и **`SetWeight: 25, ActualWeight: 25`** на шаге 1/6, а затем, после `promote --full`, та же команда показывает **шаг 6/6 `Healthy`**. `Rollout`, который сразу уходит на 100% без видимой паузы, прогрессивную доставку не отработал. **Сохрани этот скриншот.**

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно быть готово из предыдущих лаб:**

- A k3d cluster on Kubernetes **1.36** (Lab 9)
  * Кластер k3d на Kubernetes **1.36** (лаба 9)
- The Helm chart at `k8s/lab10-app/` from Lab 10 — your `Deployment` template, `Service` template, `_helpers.tpl`, and `values.yaml`
  * Helm-чарт в `k8s/lab10-app/` из лабы 10 — твой шаблон `Deployment`, шаблон `Service`, `_helpers.tpl` и `values.yaml`
- ArgoCD **3.4** managing that chart through an `ApplicationSet` (Lab 13)
  * ArgoCD **3.4**, управляющий этим чартом через `ApplicationSet` (лаба 13)
- The `echo` and `health` plumbing services from Labs 9 / 13 — `health` exposes `/metrics` you'll use in the bonus
  * Сервисы-обвязки `echo` и `health` из лаб 9 и 13 — `health` отдаёт `/metrics`, которые понадобятся в бонусе
- A Prometheus instance from **Lab 8** (Docker-Compose) — see Setup for the in-cluster decision
  * Экземпляр Prometheus из **лабы 8** (Docker-Compose) — про вариант внутри кластера см. раздел «Подготовка»

**This lab adds:**

**Что добавляет эта лаба:**

- `k8s/lab10-app/templates/rollout.yaml` — **you write** (Task 2: canary strategy)
  * `k8s/lab10-app/templates/rollout.yaml` — **пишешь ты** (задача 2: канареечная стратегия)
- `k8s/lab10-app/templates/service-active.yaml` — **you write** (Task 3: blue-green active)
  * `k8s/lab10-app/templates/service-active.yaml` — **пишешь ты** (задача 3: active для сине-зелёного деплоя)
- `k8s/lab10-app/templates/service-preview.yaml` — **you write** (Task 3: blue-green preview)
  * `k8s/lab10-app/templates/service-preview.yaml` — **пишешь ты** (задача 3: preview для сине-зелёного деплоя)
- `k8s/lab10-app/templates/analysistemplate.yaml` — **you write** (Bonus)
  * `k8s/lab10-app/templates/analysistemplate.yaml` — **пишешь ты** (бонус)
- `k8s/lab10-app/values-bluegreen.yaml` — toggle for the blue-green flow (Task 3)
  * `k8s/lab10-app/values-bluegreen.yaml` — переключатель для сине-зелёного сценария (задача 3)
- `docs/LAB14.md` — your submission report
  * `docs/LAB14.md` — твой отчёт о сдаче

Course-repo plumbing for this lab:

Обвязка в репозитории курса для этой лабы:

- `labs/lab14/prometheus-pointer.md` — what to do if your Lab 8 Prometheus is still a Docker-Compose stack outside the cluster (the bonus needs an in-cluster URL). **Do not edit.**
  * `labs/lab14/prometheus-pointer.md` — что делать, если твой Prometheus из лабы 8 всё ещё живёт в Docker-Compose вне кластера (для бонуса нужен URL внутри кластера). **Не редактировать.**

By **Lab 16** you'll replace the bare in-cluster Prometheus with kube-prometheus-stack + a `ServiceMonitor` — the `AnalysisTemplate` you write here is the consumer end of that pipeline.

К **лабе 16** ты заменишь голый Prometheus в кластере на kube-prometheus-stack и `ServiceMonitor` — а `AnalysisTemplate`, который ты пишешь здесь, станет потребителем этой цепочки.

---

## Setup

Подготовка

| Component | Version | Notes |
|---|---|---|
| Argo Rollouts controller | **v1.8.4** | Released Feb 13 2026. Do NOT use `latest` |
| `kubectl argo rollouts` plugin | **v1.8.4** | Match the controller |
| Kubernetes (k3s) | `v1.36.1-k3s1` | from Lab 9 |
| Helm | `v4.1.x` | from Lab 10 |
| Prometheus | `3.x` | Lab 8's instance — see in-cluster note below |

| Компонент | Версия | Примечания |
|---|---|---|
| Контроллер Argo Rollouts | **v1.8.4** | Выпущен 13 февраля 2026. НЕ используй `latest` |
| Плагин `kubectl argo rollouts` | **v1.8.4** | Должен совпадать с контроллером |
| Kubernetes (k3s) | `v1.36.1-k3s1` | из лабы 9 |
| Helm | `v4.1.x` | из лабы 10 |
| Prometheus | `3.x` | экземпляр из лабы 8 — см. примечание про кластер ниже |

```bash
kubectl get nodes                  # 3-node k3d cluster from Lab 9 must be Running
helm version                       # 4.1.x from Lab 10
kubectl argo rollouts version      # not installed yet — that's Task 1
```

Пояснение: первые две команды подтверждают, что кластер из лабы 9 запущен, а Helm имеет версию 4.1.x; третья покажет, что плагин ещё не установлен, — это задача 1.

> ⚠️ **Prometheus location matters for the bonus.** Lab 8 ran Prometheus in a Docker-Compose stack on your laptop. An `AnalysisTemplate` running **inside** the cluster cannot reach `http://localhost:9090` on your host. Three options, see `labs/lab14/prometheus-pointer.md`:
>
> ⚠️ **Где живёт Prometheus — важно для бонуса.** В лабе 8 Prometheus работал в стеке Docker-Compose на твоём ноутбуке. `AnalysisTemplate`, запущенный **внутри** кластера, не сможет обратиться к `http://localhost:9090` на твоей машине. Есть три варианта, см. `labs/lab14/prometheus-pointer.md`:
>
> 1. **Point at the host gateway** — k3d exposes `host.k3d.internal`; works for the bonus today but Lab 16 replaces this with the in-cluster stack anyway.
>
>    **Указать на шлюз хоста** — k3d предоставляет `host.k3d.internal`; для бонуса это сработает уже сегодня, но лаба 16 всё равно заменит этот вариант на кластерный стек.
> 2. **Run a tiny in-cluster Prometheus** in the `monitoring` namespace (manifest pointer in `labs/lab14/`).
>
>    **Поднять крошечный Prometheus внутри кластера** в namespace `monitoring` (указатель на манифест — в `labs/lab14/`).
> 3. **Skip the bonus.** Main tasks 1–4 don't need Prometheus.
>
>    **Пропустить бонус.** Основным задачам 1–4 Prometheus не нужен.

Directory layout you will produce:

В итоге у тебя получится такая структура каталогов:

```
k8s/lab10-app/
├── templates/
│   ├── rollout.yaml                # YOU write — replaces deployment.yaml (Task 2)
│   ├── service-active.yaml         # YOU write (Task 3)
│   ├── service-preview.yaml        # YOU write (Task 3)
│   └── analysistemplate.yaml       # YOU write (Bonus)
├── values.yaml                     # add a `strategy.type` toggle
└── values-bluegreen.yaml           # YOU write (Task 3) — override `strategy.type`
docs/
└── LAB14.md                        # your submission report
```

Пояснение: комментарии в дереве отмечают, какие файлы пишешь ты, а какие уже есть в чарте из лабы 10.

---

## Task 1 — Argo Rollouts fundamentals (2 pts)

Задача 1 — основы Argo Rollouts (2 балла)

### 1.1 — Why a separate controller instead of a Deployment

1.1 — Зачем отдельный контроллер вместо Deployment

In `docs/LAB14.md` write **3** sentences each on:

В `docs/LAB14.md` напиши по **3** предложения на каждый пункт:

1. What signal does a `Deployment` actually use to decide a rollout succeeded — and why is that insufficient when a bug fires on 5% of requests?
   * Какой сигнал `Deployment` на самом деле использует, чтобы решить, что rollout прошёл успешно, — и почему этого недостаточно, если баг срабатывает на 5% запросов?
2. How does Argo Rollouts shift "traffic" without a traffic router (lecture 14 slide 12) — and what's the accuracy limit at 5 replicas?
   * Как Argo Rollouts перераспределяет «трафик» без трафик-роутера (лекция 14, слайд 12) — и каков предел точности при 5 репликах?
3. Which **three** new top-level fields does the `Rollout` CRD add that `Deployment` doesn't have? (Hint: lecture 14 slides 6–11 — `strategy.canary`, `strategy.blueGreen`, `analysis`/`backgroundAnalysis`.)
   * Какие **три** новых поля верхнего уровня добавляет CRD `Rollout`, которых нет у `Deployment`? (Подсказка: лекция 14, слайды 6–11 — `strategy.canary`, `strategy.blueGreen`, `analysis`/`backgroundAnalysis`.)

### 1.2 — Install the controller pinned to v1.8.4

1.2 — Установка контроллера с фиксацией версии v1.8.4

`YOUR TASK`: install the controller into the `argo-rollouts` namespace from the **v1.8.4** release manifest — not `latest`, not `main`. The plumbing release URL pattern is `https://github.com/argoproj/argo-rollouts/releases/download/<version>/install.yaml`.

ЗАДАНИЕ: установи контроллер в namespace `argo-rollouts` из манифеста релиза **v1.8.4** — не `latest` и не `main`. Шаблон URL релиза: `https://github.com/argoproj/argo-rollouts/releases/download/<version>/install.yaml`.

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f ___        # YOUR TASK: the v1.8.4 install.yaml URL
kubectl -n argo-rollouts rollout status deploy/argo-rollouts --timeout=120s
```

Пояснение: первая команда создаёт namespace, вторая применяет манифест установки (URL нужно подставить самому), третья ждёт, пока деплой контроллера станет готов.

> 💡 The reason `latest` is forbidden: a release that breaks `bluegreen` analysis (1.8.0–1.8.3 had exactly this bug — see lecture 14 slide 21) would silently ship through your students' lab. Pin the version your TAs verified.
>
> 💡 Причина, по которой `latest` запрещён: релиз, ломающий анализ `bluegreen` (у версий 1.8.0–1.8.3 был ровно такой баг — см. лекцию 14, слайд 21), тихо уедет в студенческую лабу. Фиксируй ту версию, которую проверили твои ассистенты.

### 1.3 — Install the matching kubectl plugin

1.3 — Установка подходящего плагина kubectl

`YOUR TASK`: download the **v1.8.4** plugin binary for your OS/arch, make it executable, and place it on `PATH` as `kubectl-argo-rollouts`. Verify with `kubectl argo rollouts version` — both client and server should report `v1.8.4`.

ЗАДАНИЕ: скачай бинарник плагина **v1.8.4** под свою ОС и архитектуру, сделай его исполняемым и положи в `PATH` под именем `kubectl-argo-rollouts`. Проверь командой `kubectl argo rollouts version` — и клиент, и сервер должны сообщать `v1.8.4`.

```bash
# Linux amd64 example — adapt for darwin-arm64 / windows-amd64
curl -fSL -o kubectl-argo-rollouts ___       # YOUR TASK: the v1.8.4 plugin binary URL
chmod +x kubectl-argo-rollouts
sudo mv kubectl-argo-rollouts /usr/local/bin/

kubectl argo rollouts version                # expect: v1.8.4 client + v1.8.4 server
```

Пояснение: показан пример для Linux amd64 — для darwin-arm64 или windows-amd64 URL и имя файла будут другими; последняя команда должна показать v1.8.4 и на клиенте, и на сервере.

> 💡 The plugin is **not** a `kubectl` subcommand — it's a separate binary `kubectl` shells out to. If `kubectl argo rollouts version` says "unknown command", the binary isn't on your `PATH`.
>
> 💡 Плагин — **не** подкоманда `kubectl`, а отдельный бинарник, который `kubectl` запускает. Если `kubectl argo rollouts version` говорит «unknown command», значит бинарника нет в `PATH`.

### 1.4 — Install the dashboard (optional but useful)

1.4 — Установка дашборда (необязательно, но полезно)

`YOUR TASK`: apply the v1.8.4 dashboard manifest, port-forward it to `localhost:3100`, and screenshot the empty dashboard (you'll repopulate it once Task 2 produces a rollout).

ЗАДАНИЕ: примени манифест дашборда v1.8.4, пробрось порт на `localhost:3100` и сделай скриншот пустого дашборда (данные появятся, как только задача 2 создаст rollout).

```bash
kubectl apply -n argo-rollouts -f ___        # YOUR TASK: the v1.8.4 dashboard-install.yaml URL
kubectl argo rollouts ___                    # YOUR TASK: which subcommand opens the local UI on :3100?
                                             #            (one of: dashboard, ui, serve, web)
```

Пояснение: во второй строке нужно подставить подкоманду, открывающую локальный UI на порту 3100; подходящий вариант указан в комментарии ниже.

### 1.5 — Proof of work

1.5 — Подтверждение работы

**Paste into `docs/LAB14.md`:**

**Вставь в `docs/LAB14.md`:**

- The 3 research answers from §1.1
  * Три ответа из §1.1
- `kubectl -n argo-rollouts get pods` — controller pod `Running`
  * `kubectl -n argo-rollouts get pods` — под контроллера в состоянии `Running`
- `kubectl argo rollouts version` — client + server both `v1.8.4`
  * `kubectl argo rollouts version` — клиент и сервер оба `v1.8.4`
- A screenshot of the empty dashboard at `http://localhost:3100`
  * Скриншот пустого дашборда по адресу `http://localhost:3100`

---

## Task 2 — Canary deployment (3 pts) ← **headline task**

Задача 2 — канареечный деплой (3 балла) ← **главная задача**

### 2.1 — Convert `Deployment` → `Rollout`

2.1 — Преобразование `Deployment` → `Rollout`

Your Lab 10 chart has `templates/deployment.yaml`. **Rename it** to `templates/rollout.yaml` and change the kind. Everything inside `spec.template:` stays identical — same labels, same container, same ports, same probes. The Rollout CRD reuses your `Deployment` pod template byte-for-byte. **Don't change the `Service` selector** — Argo Rollouts injects an extra `rollouts-pod-template-hash` label to distinguish stable from canary pods, but your existing selector (`app.kubernetes.io/name` + `instance`) still matches both.

В чарте лабы 10 есть `templates/deployment.yaml`. **Переименуй его** в `templates/rollout.yaml` и поменяй kind. Всё внутри `spec.template:` остаётся таким же — те же метки, тот же контейнер, те же порты, те же пробы. CRD Rollout переиспользует pod-шаблон твоего `Deployment` байт в байт. **Не трогай селектор `Service`** — Argo Rollouts добавляет метку `rollouts-pod-template-hash`, чтобы отличать стабильные поды от канареечных, но твой текущий селектор (`app.kubernetes.io/name` + `instance`) по-прежнему подходит и тем, и другим.

`YOUR TASK`: write `templates/rollout.yaml`. The skeleton below shows the parts that are **identical to your Deployment** (so you don't reinvent the wheel) and **blanks the canary `steps:` list entirely** — you write each step.

ЗАДАНИЕ: напиши `templates/rollout.yaml`. Каркас ниже показывает части, **идентичные твоему Deployment** (чтобы не изобретать заново), а список `steps:` канареечного rollout **оставлен полностью пустым** — каждый шаг пишешь ты.

```yaml
# k8s/lab10-app/templates/rollout.yaml
apiVersion: ___                                    # YOUR TASK: which apiGroup/version owns Rollout?
                                                   #            (hint: NOT apps/v1 — see lecture 14 slide 7)
kind: ___                                          # YOUR TASK
metadata:
  name: {{ include "lab10-app.fullname" . }}-web   # SAME -web suffix as Lab 10's Deployment
                                                   # → rendered as `lab10-app-web` on the default release;
                                                   # every CLI command below targets this name.
  labels:
    {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}             # keep your Lab 10 default — needs ≥ 4 for visible 25/50/75 weights
  selector:
    matchLabels:
      {{- include "lab10-app.selectorLabels" . | nindent 6 }}
  template:
    # ⬇️ IDENTICAL to your Lab 13 Deployment pod template
    metadata:
      labels:
        {{- include "lab10-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          # ⬇️ pod template body is IDENTICAL to your Lab 10 Deployment — copy it verbatim
          # ⬇️ (ports, probes, resources, env). Shown elided here for brevity.
          # See your k8s/lab10-app/templates/deployment.yaml for the body to paste.
  strategy:
    canary:
      # YOUR TASK: write the 7-element step list for a 25 → 50 → 75 → 100 canary
      # with a MANUAL pause between every weight change. Format hints below.
      #
      # Step semantics (lecture 14 slide 7):
      #   - `- setWeight: ___`           shift traffic/replica ratio to the listed percentage
      #   - `- pause: {}`                HARD manual gate — rollout waits for `promote`
      #   - `- pause: { duration: ___ }` timed pause (e.g. "30s", "2m") — rollout resumes itself
      #
      # The headline proof of this lab is a PAUSED canary at SetWeight 25 (the lab14 refs
      # quote `CanaryPauseStep` at Step 1/6). So step 1 must shift to 25% and step 2 must be
      # a MANUAL gate, not a timed pause — otherwise the rollout never sits there long
      # enough for the `kubectl argo rollouts get` capture to show `Paused`.
      #
      # Required progression: 25 → pause → 50 → pause → 75 → pause → 100
      # (the trailing 100 doesn't need a pause; the rollout marks itself Healthy at step 6/6)
      steps:
        - setWeight: ___                            # YOUR TASK
        - pause: ___                                # YOUR TASK: which pause form? (see hint above)
        - setWeight: ___                            # YOUR TASK
        - pause: ___                                # YOUR TASK
        - setWeight: ___                            # YOUR TASK
        - pause: ___                                # YOUR TASK
        - setWeight: ___                            # YOUR TASK
```

Пояснение: каркас повторяет pod-шаблон Deployment; самостоятельно заполняются apiVersion, kind и весь список steps — веса и паузы между ними продумай сам, ориентируясь на комментарии внутри файла.

**Why each step matters:**

**Почему важен каждый шаг:**

- **`setWeight: 25` at step 1** is what makes `ActualWeight: 25` show up in `kubectl argo rollouts get`. Without it your first weight is 100 and there's no progressive delivery to demo.
  * **`setWeight: 25` на шаге 1** — именно из-за него в выводе `kubectl argo rollouts get` появляется `ActualWeight: 25`. Без него первый вес сразу 100, и демонстрировать прогрессивную доставку нечего.
- **`pause: {}` (no duration)** sits there until you run `promote`. That's the gate a real team would put metric review behind.
  * **`pause: {}` (без длительности)** висит, пока ты не выполнишь `promote`. Именно за такой гейт настоящая команда ставит проверку метрик.
- **`pause: { duration: 30s }`** is the alternative — useful for CI demos that can't wait for a human. Either is acceptable, but **at least step 2 must be `pause: {}`** so the canary visibly sits at 25% for the capture.
  * **`pause: { duration: 30s }`** — альтернатива, удобная для CI-демо, где нельзя ждать человека. Допустимо и то, и другое, но **как минимум шаг 2 обязан быть `pause: {}`**, чтобы канарейка заметно постояла на 25% для скриншота.
- **Trailing 100 with no pause** — the controller marks the rollout `Healthy` automatically; an explicit `setWeight: 100` at the end is fine but a redundant `- pause: {}` after it leaves the rollout stuck.
  * **Финальные 100 без паузы** — контроллер сам помечает rollout как `Healthy`; явный `setWeight: 100` в конце допустим, а вот лишняя `- pause: {}` после него оставит rollout висеть.

### 2.2 — Trigger the first rollout and capture the pause

2.2 — Запуск первого rollout и фиксация паузы

Apply the chart (via ArgoCD sync if you wired Lab 13's ApplicationSet, otherwise `helm upgrade`). The first `Rollout` apply creates the resource; the controller marks it `Healthy` at 100% without going through the steps (steps only run for **subsequent** revisions). Now bump the image to trigger a real canary:

Примени чарт (через синхронизацию ArgoCD, если ты настроил `ApplicationSet` из лабы 13, иначе через `helm upgrade`). Первое применение `Rollout` создаёт ресурс; контроллер сразу помечает его `Healthy` на 100%, не проходя шаги (шаги работают только для **последующих** ревизий). Теперь подними тег образа, чтобы запустить настоящую канарейку:

```bash
helm upgrade lab10-app ./k8s/lab10-app --set image.tag=v2
# or commit values-dev.yaml with the new tag and let ArgoCD sync it
```

Пояснение: первая команда обновляет тег образа через Helm, вторая — альтернатива через коммит в `values-dev.yaml` и синхронизацию ArgoCD.

Then watch:

Затем смотри:

```bash
kubectl argo rollouts ___ rollout lab10-app-web --watch   # YOUR TASK: which verb shows live status?
                                                       #            (one of: get, describe, status, list)
```

Пояснение: в комментарии перечислены варианты подкоманды — выбери ту, что показывает статус в реальном времени.

`YOUR TASK`: capture the moment the rollout sits at step 1/6 with `Message: CanaryPauseStep`, `SetWeight: 25`, `ActualWeight: 25`. This is the headline artifact of Lab 14.

ЗАДАНИЕ: поймай момент, когда rollout стоит на шаге 1/6 с `Message: CanaryPauseStep`, `SetWeight: 25`, `ActualWeight: 25`. Это главный артефакт лабы 14.

### 2.3 — Promote past every gate

2.3 — Продвижение через все гейты

```bash
kubectl argo rollouts ___ lab10-app-web --full            # YOUR TASK: which verb advances past ALL remaining pauses?
                                                       #            (full = skip the manual gates AND wait for them)
```

Пояснение: флаг --full пропускает все оставшиеся ручные паузы и доводит rollout до 100%.

Then re-capture `kubectl argo rollouts get rollout lab10-app-web` showing **step 6/6, `Status: ✔ Healthy`, `SetWeight: 100`**.

Затем снова сними вывод `kubectl argo rollouts get rollout lab10-app-web`, показывающий **шаг 6/6, `Status: ✔ Healthy`, `SetWeight: 100`**.

### 2.4 — Abort mid-rollout

2.4 — Прерывание rollout на середине

`YOUR TASK`: trigger a third rollout (bump the tag to `v3`), wait for it to reach step 1/6 paused at 25%, and then abort it. Confirm traffic returns to the stable ReplicaSet and the rollout reports `Degraded`.

ЗАДАНИЕ: запусти третий rollout (подними тег до `v3`), дождись, пока он дойдёт до шага 1/6 и встанет на паузу на 25%, а затем прерви его. Убедись, что трафик вернулся на стабильный ReplicaSet, а rollout сообщает `Degraded`.

```bash
helm upgrade lab10-app ./k8s/lab10-app --set image.tag=v3
# wait for the paused 25% step:
kubectl argo rollouts get rollout lab10-app-web

kubectl argo rollouts ___ lab10-app-web                   # YOUR TASK: which verb cancels the in-progress rollout?
kubectl argo rollouts get rollout lab10-app-web           # expect: Status: ✖ Degraded, traffic on stable

# Now resume from step 0 once you've "fixed" the image (e.g. re-bump back to v2):
kubectl argo rollouts ___ rollout lab10-app-web           # YOUR TASK: which verb restarts a Degraded rollout?
```

Пояснение: обновляем образ до v3, ждём паузу на 25%, отменяем rollout, проверяем статус, а затем после «исправления» образа запускаем rollout заново.

### 2.5 — Proof of work

2.5 — Подтверждение работы

**Paste into `docs/LAB14.md`:**

**Вставь в `docs/LAB14.md`:**

- The contents of your `templates/rollout.yaml` `spec.strategy.canary` block (just the `steps:` list — 8–10 lines)
  * Содержимое блока `spec.strategy.canary` из твоего `templates/rollout.yaml` (только список `steps:` — 8–10 строк)
- The **headline capture** from §2.2 — `kubectl argo rollouts get rollout lab10-app-web` showing `Paused`, `CanaryPauseStep`, `Step: 1/6`, `SetWeight: 25, ActualWeight: 25`
  * **Главный скриншот** из §2.2 — `kubectl argo rollouts get rollout lab10-app-web` с `Paused`, `CanaryPauseStep`, `Step: 1/6`, `SetWeight: 25, ActualWeight: 25`
- The **`promote --full`** capture from §2.3 — same command, now `Step: 6/6, Status: ✔ Healthy, SetWeight: 100`
  * **Скриншот `promote --full`** из §2.3 — та же команда, теперь `Step: 6/6, Status: ✔ Healthy, SetWeight: 100`
- The **abort + retry** captures from §2.4 — `Degraded` after abort, then `Healthy` after `retry`
  * **Скриншоты abort + retry** из §2.4 — `Degraded` после прерывания, затем `Healthy` после `retry`
- 2–3 sentences on what you observed in the dashboard during the paused window — was 1 of 4 replicas on the new ReplicaSet? (lecture 14 slide 8)
  * 2–3 предложения о том, что ты видел в дашборде во время паузы, — была ли 1 из 4 реплик на новом ReplicaSet? (лекция 14, слайд 8)

---

## Task 3 — Blue-green deployment (3 pts)

Задача 3 — сине-зелёный деплой (3 балла)

### 3.1 — Why a second strategy

3.1 — Зачем вторая стратегия

In `docs/LAB14.md` answer in 2–3 sentences:

В `docs/LAB14.md` ответь в 2–3 предложениях:

- What's the canonical workload type for blue-green over canary? (Hint: lecture 14 slide 5 — schema migrations.)
  * Для какого типа нагрузки канонически выбирают blue-green, а не canary? (Подсказка: лекция 14, слайд 5 — миграции схемы.)
- During the cutover window, how many copies of your pods are running? Why does that matter for cluster cost?
  * Сколько копий твоих подов работает в момент переключения? Почему это важно с точки зрения затрат на кластер?

### 3.2 — Write the two `Service` manifests from scratch

3.2 — Написать два манифеста `Service` с нуля

Blue-green needs **two** Services pointing at the same pod label set — Argo Rollouts rewrites their selectors on promotion so `activeService` always points at the live ReplicaSet and `previewService` always points at the staged one. Your Lab 10 chart had **one** Service; you now write **two**.

Сине-зелёному деплою нужны **два** Service, указывающих на один и тот же набор меток подов, — Argo Rollouts переписывает их селекторы при продвижении, так что `activeService` всегда указывает на живой ReplicaSet, а `previewService` — на подготовленный. В чарте лабы 10 был **один** Service; теперь ты пишешь **два**.

`YOUR TASK`: write `templates/service-active.yaml` and `templates/service-preview.yaml` from scratch. They must:

ЗАДАНИЕ: напиши `templates/service-active.yaml` и `templates/service-preview.yaml` с нуля. Они должны:

- Use the chart's `selectorLabels` helper (so the rollout controller can mutate the selector)
  * Использовать helper `selectorLabels` из чарта (чтобы контроллер rollout мог менять селектор)
- Differ only in `metadata.name` — one is `<fullname>` (the existing name your callers use), the other is `<fullname>-preview`
  * Отличаться только `metadata.name` — один `<fullname>` (существующее имя, которым пользуются клиенты), другой `<fullname>-preview`
- Both expose the same `port` / `targetPort` as your Lab 10 Service
  * Оба должны отдавать тот же `port` / `targetPort`, что и Service из лабы 10

(No skeleton — you've written ~6 Service manifests by now in Labs 9, 10, 13. If you need the API shape, `kubectl explain service.spec` is one shell command away.)

(Каркаса не будет — к этому моменту ты написал около шести манифестов Service в лабах 9, 10 и 13. Если нужна форма API, `kubectl explain service.spec` — одна команда в шелле.)

> 💡 **Naming matters.** The `activeService` name in your `Rollout` spec must match an existing `Service` *exactly*. A typo here results in `Rollout` errors like `service "app-python-actv" not found` — and the controller will not heal it. Reference the value via `include "lab10-app.fullname" .` from the same helper your Service uses.
>
> 💡 **Имя имеет значение.** Имя `activeService` в спецификации `Rollout` должно **точно** совпадать с существующим `Service`. Опечатка приведёт к ошибкам `Rollout` вида `service "app-python-actv" not found`, и контроллер это не исправит. Бери значение через `include "lab10-app.fullname" .` из того же helper, что и в Service.

### 3.3 — Configure the blue-green strategy

3.3 — Настройка сине-зелёной стратегии

`YOUR TASK`: create `values-bluegreen.yaml` that **overrides** your Rollout's `strategy:` block. Use a single boolean toggle in `values.yaml` (e.g. `strategy.type: canary` default; override to `bluegreen` in the new file) and a `{{- if eq .Values.strategy.type "bluegreen" }}` branch in `rollout.yaml`. **Don't** ship both strategies side-by-side — Argo Rollouts rejects a Rollout that declares both `canary` and `blueGreen`.

ЗАДАНИЕ: создай `values-bluegreen.yaml`, который **переопределяет** блок `strategy:` твоего Rollout. Используй один булев переключатель в `values.yaml` (например, `strategy.type: canary` по умолчанию; переопредели на `bluegreen` в новом файле) и ветку `{{- if eq .Values.strategy.type "bluegreen" }}` в `rollout.yaml`. **Не** включай обе стратегии одновременно — Argo Rollouts отвергнет Rollout, где объявлены и `canary`, и `blueGreen`.

The blue-green outer shape below is the only part shown; the **four values** that define your cutover behaviour are blanks.

Ниже показана только внешняя форма blue-green; **четыре значения**, задающие поведение переключения, оставлены пустыми.

```yaml
# in rollout.yaml, inside spec:
  strategy:
    {{- if eq .Values.strategy.type "bluegreen" }}
    blueGreen:
      activeService: ___                # YOUR TASK: name of the Service production traffic hits
                                        #            (hint: matches your <fullname>)
      previewService: ___               # YOUR TASK: name of the Service QA / smoke tests hit
                                        #            (hint: matches <fullname>-preview)
      autoPromotionEnabled: ___         # YOUR TASK: true or false?
                                        #            (this lab needs manual promote — see lecture 14 slide 9)
      scaleDownDelaySeconds: ___        # YOUR TASK: how many seconds to keep the OLD ReplicaSet
                                        #            alive after promotion, for instant `undo`?
                                        #            (suggested: 300 = 5 min — short enough to not pay
                                        #             for two ReplicaSets all day, long enough that you
                                        #             can `undo` if you spot a bug in the first minute)
      # prePromotionAnalysis:           # optional — covered in the bonus
    {{- else }}
    canary:
      steps:
        # ... your Task 2 step list ...
    {{- end }}
```

Пояснение: каркас задаёт внешний вид blueGreen; четыре значения — activeService, previewService, autoPromotionEnabled и scaleDownDelaySeconds — подбери сам, ориентируясь на комментарии.

### 3.4 — Run the blue-green cutover

3.4 — Проведение сине-зелёного переключения

```bash
helm upgrade lab10-app ./k8s/lab10-app -f values-bluegreen.yaml --set image.tag=v1
# Confirm the active Service serves v1:
kubectl port-forward svc/lab10-app-web 8080:80 &
curl -s localhost:8080/ | jq .service

# Bump to v2 — the controller spins up the new ReplicaSet behind the PREVIEW Service:
helm upgrade lab10-app ./k8s/lab10-app -f values-bluegreen.yaml --set image.tag=v2

# In another shell, validate v2 via the preview Service (prod still on v1):
kubectl port-forward svc/lab10-app-web-preview 8081:80 &
curl -s localhost:8081/ | jq .service       # YOUR TASK: confirm v2

# Confirm the two Services have DIFFERENT pod IPs in their endpoints right now —
# this is the proof the controller wrote two different selectors:
kubectl get endpoints lab10-app-web lab10-app-web-preview -o wide

# Promote — the active Service flips to v2 instantly (no new pods scheduled):
kubectl argo rollouts ___ lab10-app-web                  # YOUR TASK: same verb as Task 2 manual gate

# Test instant rollback BEFORE scaleDownDelaySeconds expires:
kubectl argo rollouts ___ lab10-app-web                  # YOUR TASK: which verb rolls back to the previous ReplicaSet?
kubectl get endpoints lab10-app-web                   # active flipped back to v1 pod IPs
```

Пояснение: сначала проверяем, что active Service отдаёт v1, затем через preview Service убеждаемся, что v2 работает, сравниваем IP подов в endpoints двух сервисов и только потом продвигаем и проверяем мгновенный откат.

### 3.5 — Proof of work

3.5 — Подтверждение работы

**Paste into `docs/LAB14.md`:**

**Вставь в `docs/LAB14.md`:**

- The two Service manifests in full (one per code block)
  * Оба манифеста Service полностью (каждый в своём блоке кода)
- Your `blueGreen:` block with all four values filled in + a one-sentence justification for `scaleDownDelaySeconds`
  * Твой блок `blueGreen:` со всеми четырьмя значениями + одно предложение с обоснованием `scaleDownDelaySeconds`
- The **two `kubectl get endpoints`** captures from §3.4 showing `lab10-app-web` and `lab10-app-web-preview` pointing at **different pod IPs** during the cutover window — this is the headline blue-green artifact
  * **Два скриншота `kubectl get endpoints`** из §3.4, показывающих, что `lab10-app-web` и `lab10-app-web-preview` указывают на **разные IP подов** в момент переключения, — это главный артефакт blue-green
- The `promote` + `undo` captures showing the active Service IPs flipping
  * Скриншоты `promote` и `undo`, показывающие смену IP активного Service
- 2 sentences comparing the speed of blue-green `undo` vs canary `abort` — which is faster, and why
  * 2 предложения о том, что быстрее — `undo` у blue-green или `abort` у canary, и почему

---

## Task 4 — Documentation (2 pts)

Задача 4 — документация (2 балла)

`YOUR TASK`: write `docs/LAB14.md` with these sections, in order:

ЗАДАНИЕ: напиши `docs/LAB14.md` со следующими разделами по порядку:

1. **Setup** — Task 1 captures (controller / plugin / dashboard versions, the three Rollout-vs-Deployment differences)
   * **Setup** — скриншоты из задачи 1 (версии контроллера, плагина и дашборда, три отличия Rollout от Deployment)
2. **Canary strategy** — your `steps:` list, the §2.5 captures (Paused at 25% → Healthy at 6/6 → Degraded after abort → Healthy after retry), and one paragraph on what the dashboard showed during the paused window
   * **Canary strategy** — твой список `steps:`, скриншоты из §2.5 (Paused на 25% → Healthy на 6/6 → Degraded после прерывания → Healthy после retry) и один абзац о том, что показывал дашборд во время паузы
3. **Blue-green strategy** — your two Service manifests, your `blueGreen:` block, the endpoint-IP diff capture, the promote + undo evidence
   * **Blue-green strategy** — твои два манифеста Service, твой блок `blueGreen:`, скриншот с разными IP в endpoints, подтверждение promote и undo
4. **Strategy comparison** — when do you reach for canary vs blue-green? Reference the lecture 14 slide 5 table; pick `app-python` (your service) and justify which strategy you'd use *in production* for a typical feature release vs a schema migration
   * **Strategy comparison** — когда выбирать canary, а когда blue-green? Сошлись на таблицу со слайда 5 лекции 14; возьми `app-python` (твой сервис) и обоснуй, какую стратегию ты выбрал бы *в продакшене* для обычного релиза фичи и для миграции схемы
5. **CLI reference** — the **five verbs** you ran (`get`, `promote`, `abort`, `retry`, `undo` — plus `version` and `dashboard`), one sentence per verb on what it does
   * **CLI reference** — **пять команд**, которые ты запускал (`get`, `promote`, `abort`, `retry`, `undo`, а также `version` и `dashboard`), по одному предложению о том, что делает каждая
6. **Challenges & learnings** — at least one real one (label selector surprise, scaleDownDelay too short, wrong apiVersion, …) — see Common Pitfalls
   * **Challenges & learnings** — хотя бы одна настоящая сложность (сюрприз с селектором меток, слишком маленький scaleDownDelay, неверный apiVersion, …) — см. раздел «Частые ошибки»

---

## Bonus Task — Metric-driven auto-abort (2 pts)

Бонусная задача — автоматический откат по метрике (2 балла)

**Objective:** Add an `AnalysisTemplate` that queries Prometheus and **auto-aborts** the canary when the metric regresses — no human intervention.

**Цель:** добавить `AnalysisTemplate`, который опрашивает Prometheus и **автоматически прерывает** канарейку при регрессии метрики — без участия человека.

The lecture (slides 10–11) showed the *shape* of an AnalysisTemplate. You write the actual query, success condition, failure limit, and cadence. Use the metrics you instrumented in **Lab 8** — `app_requests_total{status}` is the canonical RED-error-rate substrate.

В лекции (слайды 10–11) показана *форма* AnalysisTemplate. Запрос, условие успеха, лимит ошибок и частоту проверки пишешь ты. Используй метрики, которые инструментировал в **лабе 8**, — `app_requests_total{status}` это каноническая основа для RED-метрики доли ошибок.

### Bonus.1 — Write the AnalysisTemplate

Бонус 1 — написать AnalysisTemplate

`YOUR TASK`: write `k8s/lab10-app/templates/analysistemplate.yaml`. The kind + outer `spec.metrics:` shape is given; the **query, success condition, failure limit, and interval** are yours.

ЗАДАНИЕ: напиши `k8s/lab10-app/templates/analysistemplate.yaml`. kind и внешняя форма `spec.metrics:` даны; **запрос, условие успеха, лимит ошибок и интервал** — твои.

```yaml
# k8s/lab10-app/templates/analysistemplate.yaml
apiVersion: argoproj.io/v1alpha1
kind: ___                                          # YOUR TASK: which kind? (hint: slide 10)
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: ___                                # YOUR TASK: how often to sample?
                                                   #   Rule from slide 18: ≥ scrape_interval × 4
                                                   #   so if Prometheus scrapes every 15s, 60s is the floor.
      count: ___                                   # YOUR TASK: how many samples before declaring success?
                                                   #   3–5 is typical — fewer = jittery, more = slow gates.
      successCondition: ___                        # YOUR TASK: a Go expression over `result` (slide 10)
                                                   #   You're computing "fraction of non-5xx requests".
                                                   #   The query below returns a single float in [0, 1].
                                                   #   What threshold makes sense for 99.x% availability?
                                                   #   (NOT 0.95 — that's the slide-16 anti-pattern.)
      failureLimit: ___                            # YOUR TASK: how many failed samples before AUTO-ABORT?
                                                   #   0 = abort on first miss (too jittery)
                                                   #   2 = the lecture's recommendation
      provider:
        prometheus:
          address: ___                             # YOUR TASK: in-cluster Prometheus URL
                                                   #   See labs/lab14/prometheus-pointer.md for the three options.
                                                   #   Format: http://<service-name>.<namespace>:9090
          query: |
            ___                                    # YOUR TASK: PromQL for the
                                                   # "fraction of non-5xx requests" metric over a 2-minute window.
                                                   #
                                                   # Use the Lab 8 metric: app_requests_total{status="..."}
                                                   # The shape is:
                                                   #   sum(rate(<good>[2m])) / sum(rate(<all>[2m]))
                                                   # — see slide 10 for the literal template.
                                                   #
                                                   # IMPORTANT: when the canary pod has just started, no requests
                                                   # have hit it yet → the numerator and denominator are both 0
                                                   # → Prometheus returns an EMPTY VECTOR → AnalysisTemplate
                                                   # crashes on `result[0]` with NaN/index-out-of-range.
                                                   # Fix: append `or on() vector(0)` so the empty case returns 0
                                                   # rather than nothing. (See Common Pitfalls.)
                                                   #
                                                   # The `service-name` arg is yours to use:
                                                   # filter by `service="{{args.service-name}}"` or by the
                                                   # `app.kubernetes.io/name` label your `web` pods carry.
```

Пояснение: каркас фиксирует структуру спецификации; самостоятельно заполняются kind, interval, count, successCondition, failureLimit, адрес Prometheus и сам запрос PromQL.

### Bonus.2 — Wire it into the canary

Бонус 2 — подключить шаблон к канарейке

`YOUR TASK`: edit your Task 2 canary `steps:` so an `analysis` step fires after the first weight change, **and** add a spec-level continuous analysis so a mid-step regression aborts immediately (not just at the next gate).

ЗАДАНИЕ: отредактируй список `steps:` канарейки из задачи 2, чтобы шаг `analysis` срабатывал после первого изменения веса, **и** добавь анализ уровня спецификации, чтобы регрессия посреди шага прерывала rollout немедленно (а не на следующем гейте).

```yaml
  strategy:
    canary:
      ___:                                         # YOUR TASK: which spec-level key runs analysis CONTINUOUSLY
                                                   #   (in parallel with steps), so a regression mid-step
                                                   #   aborts immediately? (hint: lecture 14 slide 11 "pro tip")
        templates:
          - templateName: ___                      # YOUR TASK: which AnalysisTemplate name from Bonus.1?
        args:
          - name: service-name
            value: ___                             # YOUR TASK: which service name does the PromQL filter on?
      steps:
        - setWeight: 25
        - ___:                                     # YOUR TASK: which step type runs analysis as a GATE
                                                   #   (i.e. blocks until success/failure)?
            templates:
              - templateName: ___                  # YOUR TASK
            args:
              - name: service-name
                value: ___                         # YOUR TASK
        # … rest of your Task 2 steps …
```

Пояснение: нужно выбрать ключ уровня spec, запускающий анализ непрерывно, и тип шага, который ждёт результат анализа как гейт.

### Bonus.3 — Prove auto-abort

Бонус 3 — доказать автоматический откат

`YOUR TASK`: ship a deliberately broken image (e.g. add an `app.before_request` handler that returns `500` half the time) and trigger a rollout. Without touching the CLI, the rollout must go `Degraded` within `interval × failureLimit + count × interval` seconds. Capture:

ЗАДАНИЕ: выкати заведомо сломанный образ (например, добавь обработчик `app.before_request`, который в половине случаев возвращает `500`) и запусти rollout. Не трогая CLI, rollout должен перейти в `Degraded` за `interval × failureLimit + count × interval` секунд. Зафиксируй:

- `kubectl argo rollouts get rollout lab10-app-web` showing `Degraded` with `Message: RolloutAborted` after analysis failure
  * Вывод `kubectl argo rollouts get rollout lab10-app-web` со статусом `Degraded` и `Message: RolloutAborted` после провала анализа
- `kubectl get analysisruns -o wide` showing the failed `AnalysisRun` with its `Phase: Failed` and the failed `MeasurementCount`
  * Вывод `kubectl get analysisruns -o wide` с неудачным `AnalysisRun`, его `Phase: Failed` и проваленным `MeasurementCount`

### Bonus.4 — Go further (required for the full 2 pts) — pick ONE

Бонус 4 — пойти дальше (обязательно для полных 2 баллов) — выбери ОДИН вариант

**Option A — Multiple AnalysisTemplates.** Write a second AnalysisTemplate (e.g. `p95-latency` using `histogram_quantile(0.95, ...)` over your Lab 8 Histogram), reference **both** templates in a single `analysis:` step, and document how `failureLimit` interacts when *either* template fails. (One AnalysisTemplate failing aborts the rollout — the failure is OR'd across templates within a step.)

**Вариант A — несколько AnalysisTemplates.** Напиши второй AnalysisTemplate (например, `p95-latency` с `histogram_quantile(0.95, ...)` по гистограмме из лабы 8), сошлись на **оба** шаблона в одном шаге `analysis:` и опиши, как ведёт себя `failureLimit`, если проваливается *любой* из них. (Провал одного AnalysisTemplate прерывает rollout — результаты шаблонов внутри шага объединяются через ИЛИ.)

**Option B — Multiple metrics in one AnalysisTemplate.** Add a second entry to `spec.metrics:` (e.g. error-rate AND p95 latency) inside the same `AnalysisTemplate`. Argo Rollouts evaluates each independently; the rollout aborts when *any* metric exceeds its `failureLimit`. Document the cleaner ergonomics vs Option A.

**Вариант B — несколько метрик в одном AnalysisTemplate.** Добавь вторую запись в `spec.metrics:` (например, долю ошибок И p95 латентности) в тот же `AnalysisTemplate`. Argo Rollouts оценивает каждую независимо; rollout прерывается, когда *любая* метрика превышает свой `failureLimit`. Опиши, насколько это удобнее варианта A.

### Bonus.5 — Proof of work

Бонус 5 — подтверждение работы

**Paste into `docs/LAB14.md`:**

**Вставь в `docs/LAB14.md`:**

- The full `AnalysisTemplate` you wrote — query, interval, count, successCondition, failureLimit, provider URL
  * Полный `AnalysisTemplate`, который ты написал, — запрос, interval, count, successCondition, failureLimit, адрес provider
- The §Bonus.3 captures (rollout `Degraded` after analysis failure + the `AnalysisRun` showing the Prometheus measurements that triggered it)
  * Скриншоты из §Бонус.3 (rollout `Degraded` после провала анализа + `AnalysisRun` с измерениями Prometheus, которые его вызвали)
- Your Option A or B extension — second template OR second metric, with the file and the captured second `AnalysisRun`
  * Твоё расширение A или B — второй шаблон ИЛИ вторая метрика, с файлом и скриншотом второго `AnalysisRun`
- 2–3 sentences on which extension you picked and why
  * 2–3 предложения о том, какой вариант ты выбрал и почему

---

## How to Submit

Как сдавать

```bash
git switch -c lab14
git add k8s/lab10-app/templates/rollout.yaml
git add k8s/lab10-app/templates/service-active.yaml k8s/lab10-app/templates/service-preview.yaml
git add k8s/lab10-app/values-bluegreen.yaml
git add k8s/lab10-app/templates/analysistemplate.yaml  # bonus only
git add docs/LAB14.md
git commit -m "feat(lab14): argo rollouts 1.8.4 — canary + blue-green + analysis"
git push -u origin lab14
```

Пояснение: создаём ветку lab14, добавляем файлы лабы, коммитим и отправляем ветку в свой форк.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab14` → `course-repo:master` *(reviewed)*
  * `your-fork:lab14` → `course-repo:master` *(на ревью)*
- `your-fork:lab14` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab14` → `your-fork:master` *(вливается в твой собственный main по завершении)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 — controller + plugin + dashboard pinned to v1.8.4; research answers in LAB14.md
- [ ] Task 2 — Rollout written from scratch; canary 25→pause→50→pause→75→pause→100;
              PAUSED-at-25% capture + Healthy-at-6/6 capture + abort/retry captures
- [ ] Task 3 — two Service manifests written from scratch; blueGreen block with all four values;
              endpoints diff during cutover; promote + undo evidence
- [ ] Task 4 — LAB14.md with all six sections
- [ ] Bonus — AnalysisTemplate written from scratch; auto-abort demonstrated on a broken image;
              Option A or B extension documented
```

Пояснение: отмечай этот чек-лист по мере готовности — все пункты должны быть выполнены перед отправкой PR.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Fundamentals (2 pts)

Задача 1 — основы (2 балла)

- ✅ Controller, plugin, and dashboard **all v1.8.4** (not `latest`)
  * Контроллер, плагин и дашборд — **все v1.8.4** (не `latest`)
- ✅ `kubectl argo rollouts version` shows client + server both `v1.8.4`
  * `kubectl argo rollouts version` показывает и клиент, и сервер как `v1.8.4`
- ✅ The three Rollout-vs-Deployment differences documented in LAB14.md
  * Три отличия Rollout от Deployment описаны в LAB14.md

### Task 2 — Canary (3 pts)

Задача 2 — канарейка (3 балла)

- ✅ `templates/rollout.yaml` written from scratch — `apiVersion`, `kind`, full `spec.strategy.canary.steps:` list filled in
  * `templates/rollout.yaml` написан с нуля — `apiVersion`, `kind` и полный список `spec.strategy.canary.steps:` заполнены
- ✅ Step list is `25 → pause → 50 → pause → 75 → pause → 100` — **the pause after 25 must be `pause: {}` (manual)**
  * Список шагов — `25 → pause → 50 → pause → 75 → pause → 100`; **пауза после 25 должна быть `pause: {}` (ручная)**
- ✅ **Headline capture**: `kubectl argo rollouts get rollout lab10-app-web` showing `Status: ॥ Paused`, `Message: CanaryPauseStep`, `Step: 1/6`, `SetWeight: 25, ActualWeight: 25`
  * **Главный скриншот**: `kubectl argo rollouts get rollout lab10-app-web` показывает `Status: ॥ Paused`, `Message: CanaryPauseStep`, `Step: 1/6`, `SetWeight: 25, ActualWeight: 25`
- ✅ **Promotion capture**: same command after `promote --full` showing `Status: ✔ Healthy`, `Step: 6/6`, `SetWeight: 100`
  * **Скриншот продвижения**: та же команда после `promote --full` показывает `Status: ✔ Healthy`, `Step: 6/6`, `SetWeight: 100`
- ✅ Abort + retry demonstrated — captures show `Degraded` then `Healthy` again
  * Прерывание и повтор продемонстрированы — скриншоты показывают `Degraded`, затем снова `Healthy`

### Task 3 — Blue-Green (3 pts)

Задача 3 — сине-зелёный деплой (3 балла)

- ✅ `templates/service-active.yaml` and `templates/service-preview.yaml` **written from scratch** — no skeleton given in this lab
  * `templates/service-active.yaml` и `templates/service-preview.yaml` **написаны с нуля** — каркаса в этой лабе нет
- ✅ `blueGreen:` block with all four values filled in (`activeService`, `previewService`, `autoPromotionEnabled`, `scaleDownDelaySeconds`)
  * Блок `blueGreen:` со всеми четырьмя значениями (`activeService`, `previewService`, `autoPromotionEnabled`, `scaleDownDelaySeconds`)
- ✅ **Endpoints diff capture**: `kubectl get endpoints lab10-app-web lab10-app-web-preview` shows **different pod IPs** during the cutover window
  * **Скриншот разницы в endpoints**: `kubectl get endpoints lab10-app-web lab10-app-web-preview` показывает **разные IP подов** в момент переключения
- ✅ Promote + undo demonstrated — `kubectl get endpoints lab10-app-web` shows the active selector flipping back
  * Продвижение и откат продемонстрированы — `kubectl get endpoints lab10-app-web` показывает, что активный селектор вернулся обратно

### Task 4 — Documentation (2 pts)

Задача 4 — документация (2 балла)

- ✅ All six sections present in `docs/LAB14.md`
  * Все шесть разделов присутствуют в `docs/LAB14.md`
- ✅ CLI reference section names the five verbs (`get`, `promote`, `abort`, `retry`, `undo`) with a sentence each
  * Раздел справочника CLI перечисляет пять команд (`get`, `promote`, `abort`, `retry`, `undo`) с предложением о каждой
- ✅ One real challenge documented (not "I had never seen YAML before")
  * Описана одна настоящая сложность (а не «я никогда раньше не видел YAML»)

### Bonus — Metric-driven auto-abort (2 pts)

Бонус — автоматический откат по метрике (2 балла)

- ✅ `AnalysisTemplate` written from scratch — query, condition, interval, count, failureLimit all yours
  * `AnalysisTemplate` написан с нуля — запрос, условие, interval, count и failureLimit полностью твои
- ✅ Query uses the Lab 8 `app_requests_total` metric with `or on() vector(0)` guard
  * Запрос использует метрику `app_requests_total` из лабы 8 с защитой `or on() vector(0)`
- ✅ Auto-abort demonstrated on a deliberately broken image — captures show `Degraded` with `RolloutAborted` from an analysis failure (no manual `abort`)
  * Автоматическое прерывание продемонстрировано на заведомо сломанном образе — скриншоты показывают `Degraded` с `RolloutAborted` из-за провала анализа (без ручного `abort`)
- ✅ Option A or B extension shipped — second template OR second metric, with its own `AnalysisRun` evidence
  * Расширение A или B сдано — второй шаблон ИЛИ вторая метрика, со своим `AnalysisRun` в подтверждение

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Fundamentals | **2** | v1.8.4 controller + plugin + dashboard; Rollout-vs-Deployment differences |
| **Task 2** — Canary | **3** | Hand-written steps; paused-at-25% headline capture; promote-to-Healthy; abort + retry |
| **Task 3** — Blue-Green | **3** | Hand-written Services; four-value `blueGreen:` block; endpoint IPs differ during cutover; instant `undo` |
| **Task 4** — Documentation | **2** | Six sections, CLI verbs explained, real challenge |
| **Bonus** — Auto-abort | **2** | Hand-written AnalysisTemplate + Lab 8 PromQL; auto-aborted on broken image; Option A or B extension |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — основы | **2** | Контроллер, плагин и дашборд v1.8.4; отличия Rollout от Deployment |
| **Задача 2** — канарейка | **3** | Шаги написаны вручную; главный скриншот паузы на 25%; продвижение до Healthy; прерывание и повтор |
| **Задача 3** — сине-зелёный деплой | **3** | Service написаны вручную; блок `blueGreen:` с четырьмя значениями; IP в endpoints различаются при переключении; мгновенный `undo` |
| **Задача 4** — документация | **2** | Шесть разделов, объяснение команд CLI, настоящая сложность |
| **Бонус** — автоматический откат | **2** | AnalysisTemplate написан вручную + PromQL из лабы 8; автоматическое прерывание на сломанном образе; расширение A или B |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading:**

**Оценивание:**

- **10/10:** Both strategies end-to-end, the headline paused-at-25% capture is unambiguous, docs are honest
  * **10/10:** обе стратегии отработаны от начала до конца, главный скриншот паузы на 25% однозначен, документация честная
- **8–9/10:** Both work, minor gaps in evidence (e.g. blue-green endpoint diff missing)
  * **8–9/10:** обе работают, но есть мелкие пробелы в доказательствах (например, нет разницы в endpoints у blue-green)
- **6–7/10:** Canary works but blue-green Services not written from scratch, OR endpoint-IP diff missing
  * **6–7/10:** канарейка работает, но Service для blue-green написаны не с нуля, ИЛИ нет разницы IP в endpoints
- **<6/10:** Step list copy-pasted from the docs without the 25→pause headline, OR Services not written
  * **<6/10:** список шагов скопирован из документации без паузы на 25%, ИЛИ Service не написаны

---

## Resources

Ресурсы

<details>
<summary>📚 Argo Rollouts documentation</summary>

Документация Argo Rollouts

- [Argo Rollouts docs](https://argoproj.github.io/argo-rollouts/) — start at *Features*
  * [Документация Argo Rollouts](https://argoproj.github.io/argo-rollouts/) — начни с раздела *Features*
- [Rollout Specification](https://argoproj.github.io/argo-rollouts/features/specification/) — every field in the CRD
  * [Rollout Specification](https://argoproj.github.io/argo-rollouts/features/specification/) — все поля CRD
- [Canary Strategy](https://argoproj.github.io/argo-rollouts/features/canary/)
  * [Canary Strategy](https://argoproj.github.io/argo-rollouts/features/canary/) — канареечная стратегия
- [Blue-Green Strategy](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
  * [Blue-Green Strategy](https://argoproj.github.io/argo-rollouts/features/bluegreen/) — сине-зелёная стратегия
- [Analysis & Progressive Delivery](https://argoproj.github.io/argo-rollouts/features/analysis/)
  * [Analysis & Progressive Delivery](https://argoproj.github.io/argo-rollouts/features/analysis/) — анализ и прогрессивная доставка
- [`kubectl argo rollouts` CLI](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts/) — every subcommand
  * [`kubectl argo rollouts` CLI](https://argoproj.github.io/argo-rollouts/generated/kubectl-argo-rollouts/kubectl-argo-rollouts/) — все подкоманды
- [Argo Rollouts v1.8.4 release](https://github.com/argoproj/argo-rollouts/releases/tag/v1.8.4) — pin this exact tag
  * [Argo Rollouts v1.8.4 release](https://github.com/argoproj/argo-rollouts/releases/tag/v1.8.4) — зафиксируй именно этот тег

</details>

<details>
<summary>📦 Course plumbing</summary>

Материалы курса

- `labs/lab14/prometheus-pointer.md` — three ways to give the in-cluster AnalysisTemplate a reachable Prometheus URL
  * `labs/lab14/prometheus-pointer.md` — три способа дать AnalysisTemplate в кластере доступный URL Prometheus
- `plumbing/health/README.md` — the `/metrics` endpoint you can also point the bonus query at
  * `plumbing/health/README.md` — эндпоинт `/metrics`, на который тоже можно нацелить бонусный запрос

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

Частые ошибки (из реальных прогонов)

- **`kubectl rollout restart deployment/lab10-app-web` does NOT work on a Rollout.** The core `kubectl rollout` subcommands target `Deployment` / `DaemonSet` / `StatefulSet`. The `Rollout` CRD is owned by Argo Rollouts, not the K8s rollout API. Use `kubectl argo rollouts restart lab10-app-web` instead — same verb, different binary. Symptom: `error: deployments.apps "lab10-app-web" not found` even though `kubectl get rollouts` shows it.
  * **`kubectl rollout restart deployment/lab10-app-web` НЕ работает с Rollout.** Встроенные подкоманды `kubectl rollout` работают с `Deployment` / `DaemonSet` / `StatefulSet`. CRD `Rollout` принадлежит Argo Rollouts, а не rollout-API Kubernetes. Используй `kubectl argo rollouts restart lab10-app-web` — та же команда, другой бинарник. Симптом: `error: deployments.apps "lab10-app-web" not found`, хотя `kubectl get rollouts` его показывает.

- **AnalysisTemplate crashes on `result[0]` with NaN when there's no traffic.** When the canary pod has just started, no requests have hit it → `sum(rate(app_requests_total{...}[2m]))` returns an **empty vector**, not 0. The `successCondition` expression `result[0] >= 0.99` then errors out and the AnalysisRun shows `Inconclusive` or `Error` instead of evaluating. Fix: append `or on() vector(0)` to the PromQL — Prometheus then returns `0` for the empty case and the expression evaluates cleanly. This is the single most common bonus-task failure.
  * **AnalysisTemplate падает на `result[0]` с NaN, когда трафика нет.** Когда канареечный под только запустился, запросы до него ещё не дошли → `sum(rate(app_requests_total{...}[2m]))` возвращает **пустой вектор**, а не 0. Выражение `successCondition` вида `result[0] >= 0.99` тогда падает с ошибкой, и AnalysisRun показывает `Inconclusive` или `Error` вместо оценки. Исправление: добавь в конец PromQL `or on() vector(0)` — тогда для пустого случая Prometheus вернёт `0`, и выражение вычислится нормально. Это самая частая причина провала бонусной задачи.

- **`scaleDownDelaySeconds` is a foot-gun for traffic-still-on-preview surprises.** Setting it too high (e.g. `3600`) means every blue-green cutover leaves the **old** ReplicaSet running for an hour — burning compute and confusing anyone who sees stale pods. Setting it too low (e.g. `30`) means by the time you notice a v2 bug in production, there's no v1 ReplicaSet left to `undo` to. 300 seconds (5 min) is a sane default; document your choice.
  * **`scaleDownDelaySeconds` — легко выстрелить себе в ногу.** Слишком большое значение (например, `3600`) значит, что после каждого сине-зелёного переключения **старый** ReplicaSet живёт ещё час — сжигает ресурсы и путает всех, кто видит устаревшие поды. Слишком маленькое (например, `30`) значит, что к моменту, когда ты заметишь баг v2 на проде, откатываться через `undo` уже некуда — ReplicaSet v1 удалён. 300 секунд (5 минут) — разумный дефолт; обоснуй свой выбор.

- **`kubectl set image` needs the plugin, not core kubectl.** `kubectl set image rollout/lab10-app-web web=ghcr.io/...:v2` errors with `the server doesn't have a resource type "rollout"`. Use `kubectl argo rollouts set image lab10-app-web web=ghcr.io/...:v2` instead. Or — better practice — drive image changes through Helm / GitOps and let ArgoCD sync, so the change is in Git.
  * **`kubectl set image` требует плагина, а не встроенного kubectl.** `kubectl set image rollout/lab10-app-web web=ghcr.io/...:v2` падает с `the server doesn't have a resource type "rollout"`. Используй `kubectl argo rollouts set image lab10-app-web web=ghcr.io/...:v2`. Или — что правильнее — меняй образ через Helm / GitOps и дай ArgoCD синхронизировать, чтобы изменение было в Git.

- **Prometheus not in-cluster → AnalysisTemplate can't reach it.** Lab 8's Prometheus runs in a Docker-Compose stack on your laptop. From inside k3d, `http://localhost:9090` resolves to the **pod's** localhost, not your host's. Three fixes documented in `labs/lab14/prometheus-pointer.md`: (a) `host.k3d.internal:9090` from inside k3d (works today, breaks in Lab 16 when you move on); (b) deploy a minimal Prometheus into the cluster's `monitoring` namespace; (c) wait for Lab 16's kube-prometheus-stack and skip the bonus until then.
  * **Prometheus вне кластера → AnalysisTemplate до него не дотянется.** Prometheus из лабы 8 работает в Docker-Compose на твоём ноутбуке. Изнутри k3d `http://localhost:9090` указывает на localhost **пода**, а не твоей машины. Три исправления описаны в `labs/lab14/prometheus-pointer.md`: (a) `host.k3d.internal:9090` изнутри k3d (сегодня работает, но сломается в лабе 16, когда ты переедешь); (b) развернуть минимальный Prometheus в namespace `monitoring` кластера; (c) дождаться kube-prometheus-stack в лабе 16 и пропустить бонус до тех пор.

- **Selector pollution from the Lab 13 chart.** Argo Rollouts injects `rollouts-pod-template-hash` onto pod labels so it can distinguish stable from canary. If your `selectorLabels` helper from Lab 10 includes anything beyond `name` + `instance` (e.g. `version` — which is the lecture-15 anti-pattern), the controller's mutation fights your helper and the Service ends up selecting *both* ReplicaSets. Keep `selectorLabels` to the two-key minimum (this is the same rule from Lab 10's helpers).
  * **Загрязнение селектора из чарта лабы 13.** Argo Rollouts добавляет метку `rollouts-pod-template-hash` на поды, чтобы отличать стабильные от канареечных. Если твой helper `selectorLabels` из лабы 10 включает что-то кроме `name` и `instance` (например, `version` — а это антипаттерн из лекции 15), контроллер своей мутацией конфликтует с helper, и Service в итоге выбирает **оба** ReplicaSet. Держи `selectorLabels` в минимуме из двух ключей (это то же правило, что в helpers лабы 10).

- **Promoting a manual gate before the canary is actually paused.** `kubectl argo rollouts promote lab10-app-web` returns immediately if there's no active pause — it does **not** wait for the next step. If you run it too eagerly in a script, the rollout skips your headline-capture moment and runs straight to 100%. Wait until `kubectl argo rollouts get` shows `Paused / CanaryPauseStep` before promoting.
  * **Продвижение ручного гейта раньше, чем канарейка реально встала на паузу.** `kubectl argo rollouts promote lab10-app-web` возвращается сразу, если активной паузы нет, — он **не** ждёт следующего шага. Если слишком рано запустить его в скрипте, rollout пропустит твой главный момент для скриншота и уйдёт сразу на 100%. Дождись, пока `kubectl argo rollouts get` покажет `Paused / CanaryPauseStep`, и только потом продвигай.

- **`autoPromotionEnabled: true` in blue-green silently skips the manual gate.** The default is `false`, but a lot of tutorials show `true` for CI demos. With `true`, the new ReplicaSet is promoted as soon as it goes Ready — your preview-Service-validation window is **zero seconds**. Always set this to `false` in your lab values; only ever `true` in fully automated pipelines with prePromotionAnalysis.
  * **`autoPromotionEnabled: true` в blue-green молча пропускает ручной гейт.** По умолчанию `false`, но во многих туториалах для CI-демо показано `true`. С `true` новый ReplicaSet продвигается сразу, как только станет Ready, — окно проверки через preview Service равно **нулю секунд**. Всегда ставь здесь `false` в значениях лабы; `true` — только в полностью автоматизированных пайплайнах с prePromotionAnalysis.

- **`apiVersion: apps/v1` in the Rollout.** Muscle memory from `Deployment` strikes — Rollout lives in `argoproj.io/v1alpha1`, not `apps/v1`. `kubectl apply` returns `no matches for kind "Rollout" in version "apps/v1"` and the file silently fails to apply if you're using `helm template | kubectl apply -f -`. Always double-check the first two lines.
  * **`apiVersion: apps/v1` в Rollout.** Мышечная память от `Deployment` подводит — Rollout живёт в `argoproj.io/v1alpha1`, а не в `apps/v1`. `kubectl apply` вернёт `no matches for kind "Rollout" in version "apps/v1"`, а если ты используешь `helm template | kubectl apply -f -`, файл молча не применится. Всегда перепроверяй первые две строки.

</details>

<details>
<summary>🛠️ Tools worth knowing</summary>

Полезные инструменты

- `alias kr='kubectl argo rollouts'` — saves you 18 characters every command
  * `alias kr='kubectl argo rollouts'` — экономит 18 символов в каждой команде
- [`argo-rollouts/dashboard`](http://localhost:3100) — the local UI is the cleanest way to grab Task 2 / 3 screenshots
  * [`argo-rollouts/dashboard`](http://localhost:3100) — локальный UI — самый чистый способ делать скриншоты для задач 2 и 3
- [`kubectl get analysisruns -A`](https://argoproj.github.io/argo-rollouts/features/analysis/) — every analysis run the controller ever spawned, including the failed ones the rollout aborted on
  * [`kubectl get analysisruns -A`](https://argoproj.github.io/argo-rollouts/features/analysis/) — все AnalysisRun, когда-либо созданные контроллером, включая неудачные, из-за которых rollout прервался

</details>

---

## Looking Ahead

Что дальше

| Lab | What it adds to this stack |
|---:|---|
| 15 | StatefulSets — stateful workloads don't fit the Rollout model; ordered pod identity + per-pod PVC instead |
| 16 | **kube-prometheus-stack** + `ServiceMonitor` CRDs — replaces your hand-rolled Prometheus URL with a proper in-cluster monitoring stack; your `AnalysisTemplate` from this lab keeps working, just with `address: http://prometheus-operated.monitoring:9090` |

| Лаба | Что она добавляет в этот стек |
|---:|---|
| 15 | StatefulSets — рабочие нагрузки с состоянием не вписываются в модель Rollout; вместо этого упорядоченная идентичность подов + отдельный PVC на под |
| 16 | **kube-prometheus-stack** + CRD `ServiceMonitor` — заменяет твой самодельный URL Prometheus нормальным кластерным стеком мониторинга; твой `AnalysisTemplate` из этой лабы продолжает работать, просто с `address: http://prometheus-operated.monitoring:9090` |

---

**Good luck!** 🚦

**Удачи!** 🚦

> **Remember:** a `Rollout` is a `Deployment` plus a strategy. Without metric gates you've only slowed down a bad deploy; with them, you've automated the rollback. The headline of this lab — `SetWeight: 25, ActualWeight: 25, Message: CanaryPauseStep` — is the moment "deployed" stops meaning "released" and starts meaning "released *if the canary survives the gate*."
>
> **Запомни:** `Rollout` — это `Deployment` плюс стратегия. Без метрических гейтов ты лишь замедлил плохой деплой; с ними — автоматизировал откат. Главный момент этой лабы — `SetWeight: 25, ActualWeight: 25, Message: CanaryPauseStep` — это точка, где «задеплоено» перестаёт означать «выпущено» и начинает означать «выпущено, *если канарейка пройдёт гейт*».
