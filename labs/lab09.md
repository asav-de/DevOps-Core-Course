# Lab 9 — Kubernetes Fundamentals on k3d

Лаба 9 — Основы Kubernetes на k3d

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Kubernetes-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Kubernetes%201.36%20%7C%20k3d-informational)

> **Goal:** stand up a local k3d cluster (1 server + 2 agents), deploy **two** services side by side — your Lab 2 Python image and the course's Go `echo` plumbing — and prove they can talk to each other through `Service` + kube-DNS.
> **Deliverable:** a PR from `lab09` adding `k8s/` (Deployment + Service manifests for both services + the bonus Ingress) and `k8s/docs/LAB09.md` with the evidence captures.
>
> **Цель:** поднять локальный кластер k3d (1 сервер + 2 агента), развернуть **два** сервиса рядом — твой Python-образ из лабы 2 и учебный Go-сервис `echo` — и доказать, что они видят друг друга через `Service` + kube-DNS.
> **Что сдаём:** PR из ветки `lab09`, в котором добавлен каталог `k8s/` (манифесты Deployment + Service для обоих сервисов и бонусный Ingress) и `k8s/docs/LAB09.md` со скриншотами-доказательствами.

---

## Overview

Обзор

In this lab you will practice:

В этой лабе ты отработаешь:

- Driving a cluster with `kubectl get / describe / logs / exec / apply / rollout`
  * Работу с кластером через `kubectl get / describe / logs / exec / apply / rollout`
- **Writing Deployment + Service manifests from the API up** — picking the right `apiVersion/kind`, getting the label/selector triangle right, sizing requests/limits, wiring probes at a cheap endpoint
  * **Написание манифестов Deployment + Service от самого API** — правильный выбор `apiVersion/kind`, корректный «треугольник» меток и селекторов, подбор requests/limits, пробы на «дешёвом» эндпоинте
- Reading the K8s networking model: `Service` virtual IP → kube-DNS short name → endpoints list → pod IPs
  * Понимание сетевой модели K8s: виртуальный IP `Service` → короткое имя в kube-DNS → список endpoints → IP подов
- Importing locally-built images into a k3d cluster's containerd
  * Импорт локально собранных образов в containerd кластера k3d
- Scaling, rolling out, and rolling back a `Deployment`
  * Масштабирование, rolling update и откат `Deployment`

> ⚠️ **Scope:** plain YAML only — Helm comes in Lab 10. No persistent volumes, no secrets, no metrics scraping yet — those come in Labs 11, 12, 16. Don't reach for `kubectl edit` and don't templatize anything; learn the raw API first.
>
> ⚠️ **Границы лабы:** только чистый YAML — Helm будет в лабе 10. Ни persistent volumes, ни секретов, ни сбора метрик — это в лабах 11, 12 и 16. Не хватайся за `kubectl edit` и ничего не шаблонизируй: сначала выучи сырой API.

> 🪨 **Pedagogical core.** Through Lab 8 you ran **one** service. With one service, "why a stable IP and DNS name?" is genuinely hard to answer. This lab runs **two**. From inside your `web` pod you will `curl http://echo/ping` and get back `pong` — that single round-trip exercises kube-DNS, label selectors, and Service load-balancing all at once. **If you only take one capture, take that one.**
>
> 🪨 **Педагогическая суть.** До лабы 8 включительно у тебя был **один** сервис. С одним сервисом на вопрос «зачем стабильный IP и DNS-имя?» честно ответить трудно. Здесь сервисов **два**. Изнутри пода `web` ты выполнишь `curl http://echo/ping` и получишь `pong` — этот один обмен задействует сразу kube-DNS, селекторы меток и балансировку нагрузки в Service. **Если делать только один скриншот, делай именно этот.**

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно остаться с прошлых лаб:**

- `app_python/` from Lab 1 — Flask/FastAPI service with `/` and `/health`
  * `app_python/` из лабы 1 — сервис на Flask/FastAPI с эндпоинтами `/` и `/health`
- A working container image of it from Lab 2 (pushed to GHCR / Docker Hub, **or** built locally and tagged)
  * Рабочий образ этого сервиса из лабы 2 (запушенный в GHCR / Docker Hub **или** собранный локально и протегированный)
- Familiarity with `docker compose up` (Labs 6–8) — you've felt its limits
  * Опыт работы с `docker compose up` (лабы 6–8) — ты уже почувствовал его пределы

**This lab adds:**

**Эта лаба добавляет:**

- `k8s/web-deployment.yaml` — **you write** (Deployment for your Lab 2 image)
  * `k8s/web-deployment.yaml` — **пишешь ты** (Deployment для твоего образа из лабы 2)
- `k8s/web-service.yaml` — **you write** (Service in front of `web`)
  * `k8s/web-service.yaml` — **пишешь ты** (Service перед `web`)
- `k8s/echo-deployment.yaml` — **you write** (Deployment for the course's `echo` plumbing)
  * `k8s/echo-deployment.yaml` — **пишешь ты** (Deployment для учебного сервиса `echo`)
- `k8s/echo-service.yaml` — **you write** (ClusterIP Service for `echo`)
  * `k8s/echo-service.yaml` — **пишешь ты** (ClusterIP Service для `echo`)
- `k8s/ingress.yaml` — **you write** (Traefik Ingress, bonus only)
  * `k8s/ingress.yaml` — **пишешь ты** (Ingress Traefik, только для бонуса)
- `k8s/docs/LAB09.md` — your submission report
  * `k8s/docs/LAB09.md` — твой отчёт о сдаче

By Lab 10 you re-package all of these as a single **Helm 4** chart; by Lab 13 ArgoCD deploys them via an `ApplicationSet`. So get the labels/selectors right *now* — they propagate everywhere.

К лабе 10 ты переупакуешь всё это в один чарт **Helm 4**; к лабе 13 ArgoCD будет разворачивать их через `ApplicationSet`. Поэтому наведи порядок в метках и селекторах *сейчас* — они потащатся везде.

Course-repo plumbing for this lab:

Служебные файлы репозитория курса для этой лабы:

- `plumbing/echo/` — the Go HTTP service you deploy in Task 3. **Do not modify.** The pre-built image is published by the course CI to `ghcr.io/inno-devops-labs/echo:v1`. See `plumbing/echo/README.md` for the endpoint list.
  * `plumbing/echo/` — Go HTTP-сервис, который ты разворачиваешь в задаче 3. **Не менять.** Готовый образ публикует CI курса: `ghcr.io/inno-devops-labs/echo:v1`. Список эндпоинтов — в `plumbing/echo/README.md`.

---

## Setup

Настройка

Versions used in this lab — pin these where you have a choice:

Версии, которые используются в лабе — фиксируй их, где есть выбор:

| Component | Version | Released |
|---|---|---|
| Kubernetes (k3s) | `v1.36.1-k3s1` | Apr 22 2026 (1.36 "Haru") |
| k3d | 5.7+ | — |
| kubectl | 1.36.x (match the cluster) | — |
| `echo` plumbing image | `ghcr.io/inno-devops-labs/echo:v1` | published by course CI |

| Компонент | Версия | Дата выхода |
|---|---|---|
| Kubernetes (k3s) | `v1.36.1-k3s1` | 22 апреля 2026 (1.36 «Haru») |
| k3d | 5.7+ | — |
| kubectl | 1.36.x (как в кластере) | — |
| образ сервиса `echo` | `ghcr.io/inno-devops-labs/echo:v1` | публикуется CI курса |

```bash
docker --version           # 28.x or 29.x — k3d runs k3s inside Docker
k3d version                # 5.7+
kubectl version --client   # 1.36.x — match the cluster you'll create
```

Пояснение: проверь версии клиентов до создания кластера; версия kubectl должна соответствовать версии кластера (допускается расхождение на один минор).

> ⚠️ **Do not use Docker Desktop's bundled Kubernetes.** It lags upstream and you will get version-skew surprises mid-lab.
>
> ⚠️ **Не используй встроенный Kubernetes из Docker Desktop.** Он отстаёт от апстрима, и посреди лабы тебя будут ждать сюрпризы из-за расхождения версий.

Create the directory layout (you'll fill the files yourself):

Создай структуру каталогов (файлы наполнишь сам):

```
k8s/
├── web-deployment.yaml          # YOU write this (§2)
├── web-service.yaml             # YOU write this (§2)
├── echo-deployment.yaml         # YOU write this (§3)
├── echo-service.yaml            # YOU write this (§3)
├── ingress.yaml                 # YOU write this (bonus only)
└── docs/
    └── LAB09.md                 # your submission report
```

Пояснение: манифесты лежат в корне каталога `k8s/`, а отчёт о сдаче — в `k8s/docs/`.

---

## Task 1 — Local k3d cluster (2 pts)

Задача 1 — локальный кластер k3d (2 балла)

### 1.1 — Why k3d, not minikube / kind / Docker Desktop

1.1 — Почему k3d, а не minikube / kind / Docker Desktop

In `docs/LAB09.md` answer in 2–3 sentences each:

В `docs/LAB09.md` ответь по 2–3 предложения на каждый вопрос:

1. What does k3d actually run inside Docker, and how is that different from minikube's VM-based approach?
   * Что именно k3d запускает внутри Docker и чем это отличается от подхода minikube на виртуальных машинах?
2. Why does this lab need **multi-node** (`--agents 2`)? What's visible at 3 nodes that isn't visible at 1?
   * Зачем этой лабе **многонодовый** кластер (`--agents 2`)? Что видно на трёх нодах и не видно на одной?
3. Which ingress controller and LoadBalancer implementation does k3d ship by default, and what would change in your Bonus Ingress if you swapped k3d for kind?
   * Какой ingress-контроллер и какую реализацию LoadBalancer k3d ставит по умолчанию и что изменилось бы в твоём бонусном Ingress, если заменить k3d на kind?

(Lecture 9 slide 13 + the [k3d docs](https://k3d.io/) cover this.)

(Это разбирается на слайде 13 лекции 9 и в [документации k3d](https://k3d.io/).)

### 1.2 — Create the cluster

1.2 — Создание кластера

`YOUR TASK`: bring up a cluster named `devops` with **1 server + 2 agents** on Kubernetes **1.36.1**, mapping the LoadBalancer ports `80→8080` and `443→8443` on your host so the bonus Ingress is reachable.

`YOUR TASK`: подними кластер с именем `devops` из **1 сервера + 2 агентов** на Kubernetes **1.36.1**, пробросив порты LoadBalancer `80→8080` и `443→8443` на хост, чтобы был доступен бонусный Ingress.

```bash
# YOUR TASK lines below:
#   --image:  which k3s tag = K8s 1.36.1?  (see hint after the block)
#   --agents: how many agents for a multi-node cluster?
#   first -p: host port that maps cluster :80
#   second -p: host port that maps cluster :443
k3d cluster create devops \
  --image rancher/k3s:___-k3s1 \
  --agents ___ \
  -p "___:80@loadbalancer" \
  -p "___:443@loadbalancer"
```

Пояснение: заполни пропуски — тег образа k3s, число агентов и порты хоста; подсказки перечислены в комментариях перед командой.

> 💡 k3s image tags **append** `-k3s1` to the upstream K8s version. Pick the newest `v1.36.x-k3s1` tag from [`rancher/k3s` on Docker Hub](https://hub.docker.com/r/rancher/k3s/tags).
>
> 💡 Теги образов k3s **дописывают** `-k3s1` к версии K8s из апстрима. Возьми самый свежий тег `v1.36.x-k3s1` со страницы [`rancher/k3s` на Docker Hub](https://hub.docker.com/r/rancher/k3s/tags).

### 1.3 — Verify

1.3 — Проверка

```bash
kubectl config use-context k3d-devops    # k3d sets this automatically — sanity-check
kubectl version                          # server MUST report 1.36.x; client can be ±1 minor (skew is OK)
kubectl cluster-info
kubectl get nodes -o wide                # YOUR TASK: expect exactly THREE nodes — 1 server + 2 agents
```

Пояснение: `use-context` переключает kubectl на кластер k3d, а `get nodes -o wide` должен показать ровно три ноды — один сервер и два агента.

> 💡 If `kubectl` prints a `WARNING: version difference between client (1.34) and server (1.36) exceeds the supported version skew`, it's a warning, not an error — the lab still passes. To silence it, install kubectl 1.36.x to match the cluster.
>
> 💡 Если `kubectl` выводит `WARNING: version difference between client (1.34) and server (1.36) exceeds the supported version skew`, это предупреждение, а не ошибка — лаба всё равно засчитывается. Чтобы убрать его, поставь kubectl 1.36.x под версию кластера.

### 1.4 — Proof of work

1.4 — Доказательства работы

**Paste into `docs/LAB09.md`:**

**Вставь в `docs/LAB09.md`:**

- The 3-question research answers from §1.1
  * Ответы на три вопроса из §1.1
- `kubectl version` showing client + server `1.36.x`
  * Вывод `kubectl version` с клиентом и сервером `1.36.x`
- `kubectl get nodes -o wide` showing **3** nodes, all `Ready`, all `v1.36.1+k3s1`
  * Вывод `kubectl get nodes -o wide`: **3** ноды, все `Ready`, все `v1.36.1+k3s1`
- Your `k3d cluster create` command (so the grader can re-create your environment)
  * Твоя команда `k3d cluster create` (чтобы проверяющий мог воссоздать окружение)

---

## Task 2 — Deploy your `web` service (3 pts)

Задача 2 — разверни свой сервис `web` (3 балла)

This is the manifest-writing skill. The shape below shows you which fields a `Deployment` requires; the **values** are yours to derive from the K8s API + your Lab 1/2 work.

Здесь ты осваиваешь написание манифестов. Структура ниже показывает, какие поля обязательны для `Deployment`; **значения** тебе предстоит вывести из API K8s и результатов лаб 1 и 2.

### 2.1 — The Deployment

2.1 — Манифест Deployment

`YOUR TASK`: write `k8s/web-deployment.yaml`. Three labels-and-selectors must agree across this file (Deployment metadata, `spec.selector.matchLabels`, and `spec.template.metadata.labels`) — that label triangle is the #1 K8s YAML error. The Service in §2.2 will select the same label.

`YOUR TASK`: напиши `k8s/web-deployment.yaml`. В этом файле должны совпадать три места с метками и селекторами (метаданные Deployment, `spec.selector.matchLabels` и `spec.template.metadata.labels`) — ошибка в этом «треугольнике меток» встречается в YAML для K8s чаще всего. Service из §2.2 будет выбирать поды по той же метке.

```yaml
# k8s/web-deployment.yaml
apiVersion: ___                       # YOUR TASK: which apiGroup/version owns Deployment?
kind: ___                             # YOUR TASK
metadata:
  name: ___                           # YOUR TASK: lab's headline name
  labels:
    ___: ___                          # YOUR TASK: the label the Service selector will match
spec:
  replicas: ___                       # YOUR TASK: how many web pods? (lecture 9 + Task 4 scale)
  selector:
    matchLabels:
      ___: ___                        # MUST match the pod template labels below
  strategy:
    type: ___                         # YOUR TASK: which strategy for zero-downtime updates?
    rollingUpdate:
      maxUnavailable: ___             # YOUR TASK: 0 = strict zero-downtime
      maxSurge: ___
  template:
    metadata:
      labels:
        ___: ___                      # MUST match selector.matchLabels EXACTLY
    spec:
      containers:
        - name: ___                   # YOUR TASK
          image: ___                  # YOUR TASK: your Lab 2 image. Two valid forms:
                                      #   - GHCR pull: ghcr.io/<you>/devops-info-service:1.0.0  (public package — Lab 2 Task 7)
                                      #   - Local tag: devops-info-service:lab02-multi          (must `k3d image import` — see §2.3)
          imagePullPolicy: ___        # YOUR TASK: which policy lets a k3d-imported image work?
                                      #            (hint: see Setup note about local images + Common Pitfalls)
          ports:
            - name: ___               # YOUR TASK: name this port (e.g. `http`). Lab 10's Helm chart
                                      #            and Lab 16's ServiceMonitor both target the port BY NAME.
                                      #            Pick a name now or rewrite both labs later.
              containerPort: ___      # YOUR TASK: the port your Lab 1 app listens on
              protocol: TCP
          resources:
            requests:
              cpu: ___                # YOUR TASK: scheduler reserves this — see lecture 9 slide 15
              memory: ___
            limits:
              cpu: ___                # YOUR TASK
              memory: ___             # YOUR TASK: memory-OOMKill ceiling — set tightly
          livenessProbe:
            httpGet:
              path: ___               # YOUR TASK: which endpoint from Lab 1? (hint: cheap, not the app root)
              port: ___
            initialDelaySeconds: ___  # YOUR TASK
            periodSeconds: ___
          readinessProbe:
            httpGet:
              path: ___               # YOUR TASK
              port: ___
            periodSeconds: ___        # YOUR TASK: fail fast — readiness gates traffic
```

Пояснение: пропуски `___` заполняются твоими значениями; комментарии `YOUR TASK` подсказывают, что выбрать, а комментарии с дополнительным отступом объясняют ограничения.

**Why each blank matters (read this before filling them in):**

**Почему важен каждый пропуск (прочитай перед тем, как заполнять):**

- **`apiVersion` / `kind`** — Deployment lives in `apps/v1` (the *only* GA group/version for it). `Pod` is `v1`, `Service` is `v1`. Memorize the table; you'll write it from muscle by Lab 13.
  * **`apiVersion` / `kind`** — Deployment живёт в `apps/v1` (единственная GA-группа/версия для него). `Pod` — это `v1`, `Service` — тоже `v1`. Запомни таблицу: к лабе 13 ты будешь писать это на автомате.
- **The label triangle** — `metadata.labels`, `spec.selector.matchLabels`, and `spec.template.metadata.labels` are three *separate* dicts that all need the same key/value pair. The selector is **immutable after creation**; if you mis-key it you must `kubectl delete` and re-apply.
  * **Треугольник меток** — `metadata.labels`, `spec.selector.matchLabels` и `spec.template.metadata.labels` — это три *разных* словаря, в которых должна быть одна и та же пара ключ/значение. Селектор **неизменяем после создания**; если ошибся с ключом, придётся сделать `kubectl delete` и применить заново.
- **`imagePullPolicy`** — `Always` re-pulls every restart; **fails on k3d for locally-built images** because the registry doesn't have them. `IfNotPresent` uses what's already in containerd (which is what `k3d image import` populates). `Never` is the paranoid version. Pick deliberately.
  * **`imagePullPolicy`** — `Always` заново скачивает образ при каждом перезапуске и **падает на k3d для локально собранных образов**, потому что в реестре их нет. `IfNotPresent` использует то, что уже лежит в containerd (а туда образ кладёт `k3d image import`). `Never` — параноидальный вариант. Выбирай осознанно.
- **`resources.requests`** — the **scheduler** reads these to pick a node. Set too low and noisy neighbours starve you; too high and your pods sit `Pending` because no node has the headroom.
  * **`resources.requests`** — по этим значениям **планировщик** выбирает ноду. Слишком мало — «шумные соседи» задушат твои поды; слишком много — поды будут висеть в `Pending`, потому что ни на одной ноде нет запаса.
- **`resources.limits.memory`** — past this, the kernel OOMKills the container. Set tightly so a leak shows up as a restart, not a node-wide swap death. **CPU** limits cause silent throttling — be more lenient there.
  * **`resources.limits.memory`** — при превышении ядро убивает контейнер через OOMKill. Ставь лимит плотно, чтобы утечка проявлялась как перезапуск, а не как «смерть» всей ноды из-за swap. Лимиты **CPU** вызывают незаметный троттлинг — здесь будь щедрее.
- **Probes** — `livenessProbe` failing = container restarted. `readinessProbe` failing = pod pulled from the Service endpoints list (no restart). Both pointing at the app's root `/` is the single most common production foot-gun (slow page = restart loop = outage). Pick a `/health` or `/healthz` that returns 200 with no DB calls.
  * **Пробы** — провал `livenessProbe` = контейнер перезапускается. Провал `readinessProbe` = под убирают из списка endpoints Service (без перезапуска). Если обе пробы смотрят на корень приложения `/` — это самая частая производственная ловушка (медленная страница = цикл перезапусков = недоступность). Выбирай `/health` или `/healthz`, который отдаёт 200 без обращений к БД.
- **`strategy.rollingUpdate.maxUnavailable: 0`** is what guarantees zero downtime in Task 4. The default (25%) means one of your three pods is gone during the update.
  * **`strategy.rollingUpdate.maxUnavailable: 0`** — именно это гарантирует нулевой простой в задаче 4. Значение по умолчанию (25%) означает, что во время обновления один из трёх подов будет недоступен.

### 2.2 — The Service

2.2 — Манифест Service

`YOUR TASK`: write `k8s/web-service.yaml`. Choose the **Service type** that makes the app reachable from your laptop without a port-forward (lecture 9 slide 9 has the table; on k3d it's the development-friendly one).

`YOUR TASK`: напиши `k8s/web-service.yaml`. Выбери **тип Service**, при котором приложение доступно с ноутбука без port-forward (таблица — на слайде 9 лекции 9; на k3d нужен удобный для разработки вариант).

```yaml
# k8s/web-service.yaml
apiVersion: ___                       # YOUR TASK
kind: ___                             # YOUR TASK
metadata:
  name: ___                           # YOUR TASK: kube-DNS will resolve THIS name inside the cluster
spec:
  type: ___                           # YOUR TASK: see lecture 9 slide 9 — pick the one that exposes :NodePort on every node
  selector:
    ___: ___                          # MUST match the Deployment pod labels — kube-proxy uses this to build endpoints
  ports:
    - name: ___                       # YOUR TASK: same name you gave the containerPort (e.g. `http`).
                                      #            Lab 16's ServiceMonitor uses this string, not the number.
      port: ___                       # YOUR TASK: the port the Service listens on (cluster-side)
      targetPort: ___                 # YOUR TASK: prefer the port NAME (e.g. `http`) over a number —
                                      #            keeps the Service stable when the container port changes
      protocol: TCP
      # nodePort: ___                 # OPTIONAL: pin it (30000–32767), or let K8s pick
```

Пояснение: `type` и имена портов из этого файла должны согласовываться с Deployment из §2.1 — Service находит поды по селектору, а трафик направляет на `targetPort`.

**Why each blank matters:**

**Почему важен каждый пропуск:**

- **Service `name`** — this is what kube-DNS resolves inside the cluster. `curl http://web/` from any other pod hits this Service. Pick something short and stable; renaming a Service is a footgun (DNS clients cache).
  * **Имя Service (`name`)** — именно его резолвит kube-DNS внутри кластера. `curl http://web/` из любого другого пода попадёт в этот Service. Выбирай короткое и стабильное имя; переименование Service — ловушка (DNS-клиенты кешируют).
- **`type`** — `ClusterIP` (default) is internal-only; `NodePort` exposes a high port on every node; `LoadBalancer` provisions a cloud LB (klipper-lb on k3d). For local dev a NodePort works once you've mapped the host port; `LoadBalancer` works on k3d too thanks to klipper. Pick one, justify it.
  * **`type`** — `ClusterIP` (по умолчанию) доступен только внутри; `NodePort` открывает высокий порт на каждой ноде; `LoadBalancer` создаёт облачный балансировщик (на k3d — klipper-lb). Для локальной разработки NodePort работает, если проброшен порт хоста; `LoadBalancer` на k3d тоже работает благодаря klipper. Выбери один вариант и обоснуй.
- **`port` vs `targetPort`** — `port` is what *clients* hit on the Service IP; `targetPort` is what the **container** listens on. Mismatched here = endpoints exist but no connectivity. Don't blindly set them equal — they're different concepts.
  * **`port` против `targetPort`** — `port` — это то, куда *клиенты* обращаются по IP Service; `targetPort` — то, что слушает **контейнер**. Если здесь рассинхрон, endpoints есть, а связи нет. Не приравнивай их бездумно — это разные понятия.
- **Named ports** — `ports[].name: http` (and `targetPort: http` referencing the container's `ports[].name`) is **not optional polish**. Lab 16's `ServiceMonitor.spec.endpoints[].port` is a **string** (the port *name*), not a number — a bare numeric port silently fails to resolve and the operator skips your service. Set the name now or rewrite this Service later.
  * **Именованные порты** — `ports[].name: http` (и `targetPort: http`, ссылающийся на `ports[].name` контейнера) — **не необязательная полировка**. В лабе 16 поле `ServiceMonitor.spec.endpoints[].port` имеет тип **строка** (имя порта), а не число: голый числовой порт молча не резолвится, и оператор пропускает твой сервис. Задай имя сейчас или переписывай Service потом.
- **`selector`** — must match the pod labels exactly. `kubectl get endpoints web` is your debugging oracle: if it's empty, your selector doesn't match any pod.
  * **`selector`** — должен точно совпадать с метками подов. `kubectl get endpoints web` — твой оракул отладки: если список пуст, селектор не совпал ни с одним подом.

### 2.3 — Load your image and apply

2.3 — Загрузи образ и примени манифесты

**Two paths** — pick one based on whether your Lab 2 image is public on GHCR:

**Два пути** — выбери один в зависимости от того, опубликован ли твой образ из лабы 2 в GHCR:

**Path A — GHCR (recommended).** Your `ghcr.io/<you>/devops-info-service:1.0.0` is public (per Lab 2 §7). k3d's containerd pulls it directly; **no `k3d image import` needed**. Skip to `kubectl apply`.

**Путь A — GHCR (рекомендуется).** Твой `ghcr.io/<you>/devops-info-service:1.0.0` публичен (см. §7 лабы 2). containerd в k3d скачает его напрямую; **`k3d image import` не нужен**. Переходи к `kubectl apply`.

**Path B — Local build only.** Your image only exists in your laptop's Docker (`devops-info-service:lab02-multi` or similar) — k3d's containerd has never heard of it. You **must** import it:

**Путь B — только локальная сборка.** Образ есть только в Docker на твоём ноутбуке (`devops-info-service:lab02-multi` или подобное) — containerd k3d о нём не знает. Его **обязательно** нужно импортировать:

```bash
docker images | grep devops-info-service         # confirm the tag exists locally
k3d image import ___ --cluster devops            # YOUR TASK: paste the exact `<name>:<tag>` from above
```

Пояснение: подставь точное имя и тег образа из вывода `docker images`.

> 💡 The k3d nodes run k3s in their own containers with their own containerd — your laptop's Docker daemon is invisible to them. `k3d image import` is how you bridge that gap (it `docker save | docker load`s into each k3d node). If `kubectl get pods` shows `ErrImagePull`/`ImagePullBackOff` and you're sure the tag is right, you forgot this step.
>
> 💡 Ноды k3d запускают k3s в собственных контейнерах со своим containerd — Docker-демон твоего ноутбука им не виден. `k3d image import` как раз перекидывает образ через эту границу (делает `docker save | docker load` в каждую ноду k3d). Если `kubectl get pods` показывает `ErrImagePull`/`ImagePullBackOff`, а тег точно правильный, значит ты забыл этот шаг.

Once the image is reachable (either path):

Когда образ доступен (любым из путей):

```bash
kubectl apply -f k8s/web-deployment.yaml
kubectl apply -f k8s/web-service.yaml

kubectl rollout status deploy/web              # blocks until 3/3 ready
kubectl get pods -l app=web -o wide            # YOUR TASK: confirm pods spread across all 3 nodes
kubectl get endpoints web                      # MUST list 3 pod IPs once readiness passes
```

Пояснение: `rollout status` блокируется до готовности подов, а `kubectl get endpoints web` должен показать IP всех трёх подов.

> 💡 **Debug ladder when this doesn't work first time:**
> 1. `kubectl get pods` — `ImagePullBackOff` = registry/credentials/`imagePullPolicy` wrong; `CrashLoopBackOff` = the container exits, look at logs.
> 2. `kubectl describe pod <name>` — bottom **Events** section has the real failure.
> 3. `kubectl logs <pod-name>` — your app's stdout.
> 4. `kubectl get endpoints web` — empty list = selector/label mismatch (the label triangle is wrong).
>
> 💡 **Лесенка отладки, если с первого раза не заработало:**
> 1. `kubectl get pods` — `ImagePullBackOff` = неверный реестр/учётные данные/`imagePullPolicy`; `CrashLoopBackOff` = контейнер завершается, смотри логи.
> 2. `kubectl describe pod <name>` — настоящая причина в нижней секции **Events**.
> 3. `kubectl logs <pod-name>` — stdout твоего приложения.
> 4. `kubectl get endpoints web` — пустой список = несовпадение селектора и меток (ошибка в треугольнике меток).

### 2.4 — Proof of work

2.4 — Доказательства работы

**Paste into `docs/LAB09.md`:**

**Вставь в `docs/LAB09.md`:**

- `kubectl get deploy,rs,pods,svc,endpoints -l app=web` — must show **3/3** ready and **3 endpoint IPs**
  * Вывод `kubectl get deploy,rs,pods,svc,endpoints -l app=web`: **3/3** готовых пода и **3 IP в endpoints**
- `kubectl get pods -l app=web -o wide` — the **NODE** column must show all 3 nodes represented (proves multi-node scheduling — this is half the reason you ran `--agents 2`)
  * Вывод `kubectl get pods -l app=web -o wide`: в столбце **NODE** должны быть все 3 ноды (это доказывает многонодовое планирование — половина смысла флага `--agents 2`)
- A `kubectl port-forward svc/web 8080:80` + `curl -s localhost:8080/health` capture proving the Service is wired to a working app
  * Скриншот `kubectl port-forward svc/web 8080:80` и `curl -s localhost:8080/health`, доказывающий, что Service ведёт к работающему приложению
- Your `imagePullPolicy` choice + 1 sentence justifying it
  * Твой выбор `imagePullPolicy` и одно предложение с обоснованием
- Your resource requests/limits + 1 sentence justifying them
  * Твои requests/limits и одно предложение с обоснованием

---

## Task 3 — Deploy the `echo` plumbing service (2 pts)

Задача 3 — разверни служебный сервис `echo` (2 балла)

The 2nd service. Read `plumbing/echo/README.md` for the endpoint contract — you do **not** build or modify this code.

Второй сервис. Контракт эндпоинтов описан в `plumbing/echo/README.md` — этот код ты **не** собираешь и не меняешь.

| Path | Behaviour |
|---|---|
| `GET /ping` | Returns `pong\n` — your headline smoke test in Task 4 |
| `* /echo` | JSON with `hostname` + a request counter — useful for spotting load-balancing |
| `GET /healthz` | Returns `ok` (200) — wire into probes |
| `GET /metrics` | Prometheus format (you'll scrape this in Lab 16) |

| Путь | Поведение |
|---|---|
| `GET /ping` | Возвращает `pong\n` — твой главный смоук-тест в задаче 4 |
| `* /echo` | JSON с `hostname` и счётчиком запросов — удобно замечать балансировку |
| `GET /healthz` | Возвращает `ok` (200) — подключай к пробам |
| `GET /metrics` | Формат Prometheus (будешь собирать в лабе 16) |

The image is `ghcr.io/inno-devops-labs/echo:v1`; it listens on **port 8081**.

Образ — `ghcr.io/inno-devops-labs/echo:v1`; слушает **порт 8081**.

### 3.1 — The echo Deployment

3.1 — Deployment для echo

`YOUR TASK`: write `k8s/echo-deployment.yaml`. Same shape as `web` but different labels, different image, different port, different replica count — **2 replicas** is non-negotiable because Task 4 needs the Service to round-robin across **two** pods.

`YOUR TASK`: напиши `k8s/echo-deployment.yaml`. Форма та же, что у `web`, но другие метки, образ, порт и число реплик — **2 реплики** обязательны, потому что в задаче 4 Service должен распределять трафик между **двумя** подами.

```yaml
# k8s/echo-deployment.yaml
apiVersion: ___                       # YOUR TASK
kind: ___                             # YOUR TASK
metadata:
  name: ___                           # YOUR TASK
  labels:
    ___: ___                          # YOUR TASK: must be DIFFERENT from web's label value
spec:
  replicas: ___                       # YOUR TASK: read the prose above — why two?
  selector:
    matchLabels:
      ___: ___
  template:
    metadata:
      labels:
        ___: ___
    spec:
      containers:
        - name: ___
          image: ___                  # YOUR TASK: the published echo image — see plumbing/echo/README.md
          ports:
            - name: ___               # YOUR TASK: name the port (e.g. `http`). Same reason as the web Deployment.
              containerPort: ___      # YOUR TASK: see echo README — what port?
              protocol: TCP
          resources:
            requests: { cpu: ___, memory: ___ }    # YOUR TASK: echo is a tiny Go binary
            limits:   { cpu: ___, memory: ___ }
          readinessProbe:
            httpGet:
              path: ___               # YOUR TASK: echo's readiness endpoint — README has it
              port: ___
            periodSeconds: ___
          livenessProbe:
            httpGet:
              path: ___
              port: ___
            initialDelaySeconds: ___
            periodSeconds: ___
```

Пояснение: манифест echo почти повторяет манифест `web`; меняются метки, образ, порт, число реплик и адреса проб.

> 💡 **Go binaries are tiny.** Set `requests.memory` to single-digit MiB and `requests.cpu` to ~10–50m. If you copy your `web` numbers you'll over-reserve cluster capacity and pods will go `Pending`.
>
> 💡 **Go-бинарники крошечные.** Ставь `requests.memory` в однозначные MiB, а `requests.cpu` — около 10–50m. Если скопируешь цифры из `web`, то завысишь резервы кластера, и поды уйдут в `Pending`.

### 3.2 — The echo Service

3.2 — Service для echo

`YOUR TASK`: write `k8s/echo-service.yaml`. This one is `ClusterIP` (internal-only — the `web` pod calls it; no laptop access needed). The cross-service call in §4.1 will hit `http://echo:___/ping` — pick the Service `port` that makes that URL clean (lecture 9 slide 9 has the convention).

`YOUR TASK`: напиши `k8s/echo-service.yaml`. Здесь `ClusterIP` (только внутри кластера — его вызывает под `web`, с ноутбука доступ не нужен). Межсервисный вызов в §4.1 пойдёт на `http://echo:___/ping` — выбери `port` Service так, чтобы URL был чистым (условность описана на слайде 9 лекции 9).

```yaml
# k8s/echo-service.yaml
apiVersion: ___
kind: ___
metadata:
  name: ___                           # YOUR TASK: the DNS name `web` will use to reach echo
spec:
  type: ___                           # YOUR TASK: internal-only — which type?
  selector:
    ___: ___                          # MUST match echo pod labels (NOT web's)
  ports:
    - name: ___                       # YOUR TASK: same name you gave the containerPort
      port: ___                       # YOUR TASK: HTTP-default port, so `http://echo/ping` works with no `:port`
      targetPort: ___                 # YOUR TASK: prefer the port NAME (string) over the container port number
      protocol: TCP
```

Пояснение: имя Service (`name`) станет DNS-именем, по которому `web` дотянется до echo; селектор должен совпадать с метками подов echo, а не web.

> 💡 **Why service `port: 80`?** Because `http://echo/ping` (no port) defaults to 80. If you set `port: 8081`, callers have to write `http://echo:8081/ping` — works, but the lecture/docs/your future Helm chart all assume the clean form. The Service port and container port are **decoupled**; that's the whole point.
>
> 💡 **Почему `port: 80` у Service?** Потому что `http://echo/ping` (без порта) по умолчанию идёт на 80. Если указать `port: 8081`, вызывающим придётся писать `http://echo:8081/ping` — работает, но лекция, документация и твой будущий Helm-чарт рассчитаны на чистый вариант. Порт Service и порт контейнера **развязаны** — в этом весь смысл.

### 3.3 — Apply and verify

3.3 — Примени и проверь

```bash
kubectl apply -f k8s/echo-deployment.yaml
kubectl apply -f k8s/echo-service.yaml

kubectl get pods,svc,endpoints -l app=echo     # 2 pods Ready, 2 endpoint IPs
kubectl rollout status deploy/echo
```

Пояснение: после применения обоих манифестов убедись, что оба пода в состоянии Ready, а в endpoints два IP.

### 3.4 — Proof of work

3.4 — Доказательства работы

**Paste into `docs/LAB09.md`:**

**Вставь в `docs/LAB09.md`:**

- `kubectl get pods,svc,endpoints -l app=echo` — **2** pods Ready, Service has **2** endpoints
  * Вывод `kubectl get pods,svc,endpoints -l app=echo`: **2** пода Ready, у Service **2** эндпоинта
- One `kubectl describe svc echo` capture showing the `Selector` and `Endpoints` lines
  * Один скриншот `kubectl describe svc echo` с полями `Selector` и `Endpoints`
- 1–2 sentences on **why your echo `requests` are smaller than `web`'s** — the tiny Go binary justification
  * 1–2 предложения о том, **почему `requests` у echo меньше, чем у `web`** — обоснование через крошечный Go-бинарник

---

## Task 4 — Cross-service networking, scaling, rolling updates (2 pts) ← **headline task**

Задача 4 — сетевое взаимодействие сервисов, масштабирование, rolling update (2 балла) ← **главная задача**

### 4.1 — The kube-DNS proof — the most important command in this lab

4.1 — Доказательство работы kube-DNS — самая важная команда в лабе

`YOUR TASK`: from **inside a `web` pod**, call `echo`'s `/ping` endpoint by its **DNS name** and capture `pong`. No port-forward. No NodePort. The request must traverse pod → Service DNS → kube-proxy → echo pod, with kube-DNS doing the name resolution.

`YOUR TASK`: **изнутри пода `web`** вызови эндпоинт `/ping` сервиса `echo` по его **DNS-имени** и получи `pong`. Без port-forward. Без NodePort. Запрос должен пройти pod → DNS Service → kube-proxy → под echo, а разрешением имени занимается kube-DNS.

```bash
WEBPOD=$(kubectl get pod -l ___=___ -o jsonpath=___)    # YOUR TASK: get the name of any web pod
                                                        #            (hint: -o jsonpath='{.items[0].metadata.name}')

kubectl exec $WEBPOD -- ___ ___                         # YOUR TASK: hit http://echo/ping
                                                        #            (hint: which tool? — see "If curl is missing" below)
# expected output:
# pong
```

Пояснение: первая команда сохраняет имя любого пода `web`, вторая выполняет HTTP-запрос изнутри этого пода; ожидаемый ответ — `pong`.

**Why this matters.** If you get `pong`, six things just worked at once:

**Почему это важно.** Если ты получил `pong`, одновременно сработали шесть вещей:

1. **kube-DNS** in `kube-system` resolved `echo` → the Service's ClusterIP
   * **kube-DNS** в `kube-system` разрешил `echo` → в ClusterIP сервиса
2. The Service's **selector** matched the echo pod labels
   * **Селектор** Service совпал с метками подов echo
3. **kube-proxy** programmed iptables/IPVS rules to route Service IP → one of two pod IPs
   * **kube-proxy** настроил правила iptables/IPVS, чтобы направить IP Service → на один из двух IP подов
4. The echo pod's **readinessProbe** passed, so it appears in `endpoints`
   * **readinessProbe** пода echo прошла, поэтому под попал в `endpoints`
5. Cluster-internal networking forwarded the request across the node boundary (probably — `web` and `echo` likely landed on different nodes)
   * Внутрикластерная сеть передала запрос через границу нод (скорее всего — `web` и `echo` обычно попадают на разные ноды)
6. Your manifests' **label triangle** is consistent end-to-end
   * **Треугольник меток** в твоих манифестах согласован от начала до конца

If any one of those is wrong, you get a timeout or a DNS NXDOMAIN. **Save the output.** This is the headline artifact of Lab 9.

Если хоть что-то из этого не так, ты получишь таймаут или DNS NXDOMAIN. **Сохрани вывод.** Это главный артефакт лабы 9.

<details>
<summary>💡 If your <code>web</code> image doesn't ship <code>curl</code></summary>

A slim Lab 2 image likely doesn't have `curl`. Two options — both acceptable as long as the call originates **inside the cluster**:

Тонкий образ из лабы 2, скорее всего, собран без `curl`. Два варианта — оба годятся, если запрос исходит **изнутри кластера**:

```bash
# Option A — use python (Lab 1 left you a Python interpreter)
kubectl exec $WEBPOD -- python -c \
  "import urllib.request; print(urllib.request.urlopen('http://echo/ping').read().decode())"

# Option B — throwaway client pod inside the cluster
kubectl run tmp --rm -it --image=curlimages/curl:8.11.0 --restart=Never -- \
  curl -s http://echo/ping
```

Пояснение: вариант A использует Python из образа, вариант B поднимает временный под с curl; оба выполняют запрос внутри кластера.

What does **NOT** count: `curl localhost:8088/ping` after `port-forward svc/echo` — that proves your laptop can reach echo, not that `web` can. The whole point is the **pod-to-pod** call.

Что **НЕ** считается: `curl localhost:8088/ping` после `port-forward svc/echo` — это доказывает, что с ноутбука доступен echo, а не что его видит `web`. Весь смысл — вызов **pod-to-pod**.

</details>

### 4.2 — Service load-balancing across two echo pods

4.2 — Балансировка нагрузки Service между двумя подами echo

The echo Service has 2 endpoints. Hit `/echo` a few times — the response includes the responding pod's `hostname`, so you can see kube-proxy round-robining the traffic.

У Service echo два эндпоинта. Вызови `/echo` несколько раз — в ответе есть `hostname` пода, который ответил, так что видно, как kube-proxy распределяет трафик по кругу.

```bash
kubectl exec $WEBPOD -- sh -c '___'                     # YOUR TASK: hit http://echo/echo 6+ times in a loop,
                                                        #            extract or print the hostname per call
# expected: hostname rotates between the 2 echo pods (roughly 50/50 over enough calls)
```

Пояснение: в цикле нужно несколько раз обратиться к `/echo` и вывести `hostname` из каждого ответа.

> 💡 Round-robin is per-connection by default, not per-request. If you see the same hostname every time, your client is reusing one connection — add a fresh request each time (`-H 'Connection: close'` for curl, or just loop fresh `urlopen` calls in Python).
>
> 💡 По умолчанию round-robin работает на уровне соединений, а не запросов. Если каждый раз видишь одно и то же имя, клиент переиспользует соединение — создавай новый запрос каждый раз (`-H 'Connection: close'` для curl или просто новый вызов `urlopen` в цикле на Python).

### 4.3 — Scale `web` to 5 replicas

4.3 — Масштабируй `web` до 5 реплик

`YOUR TASK`: scale up. Use the **declarative** path (edit `replicas:` in the manifest and `kubectl apply -f`) — imperative `kubectl scale` works but doesn't survive the next `apply` of your manifest, which is the GitOps-killing footgun you'll meet again in Lab 13.

`YOUR TASK`: масштабируй. Используй **декларативный** путь (поменяй `replicas:` в манифесте и выполни `kubectl apply -f`) — императивный `kubectl scale` работает, но не переживёт следующего `apply` манифеста; это та самая ловушка, убивающая GitOps, с которой ты снова столкнёшься в лабе 13.

```bash
# After editing the manifest:
kubectl apply -f k8s/web-deployment.yaml
kubectl get pods -l app=web -w           # watch new pods come up
kubectl get endpoints web                # 5 pod IPs once readiness passes
```

Пояснение: после правки манифеста примени его, дождись новых подов и убедись, что в endpoints пять IP.

### 4.4 — Rolling update + rollback

4.4 — Rolling update и откат

Change the `web` image to a different tag (rebuild your Lab 2 image with a new tag, or temporarily change `name:` of the container — anything that makes the pod template hash different). Apply and watch:

Смени тег образа `web` (пересобери образ из лабы 2 с новым тегом или временно поменяй поле `name:` у контейнера — подойдёт любое изменение, меняющее хеш шаблона пода). Примени и наблюдай:

```bash
kubectl apply -f k8s/web-deployment.yaml
kubectl rollout status deploy/web        # blocks until new ReplicaSet is fully ready
kubectl rollout history deploy/web       # shows both revisions

kubectl rollout undo deploy/web          # back to the previous revision
kubectl rollout status deploy/web
```

Пояснение: `rollout history` показывает ревизии, а `rollout undo` возвращает предыдущую; после отката снова дождись готовности подов.

With `maxUnavailable: 0` (from §2.1), at no point should `kubectl get endpoints web` drop below 5. **Verify this in another shell.**

При `maxUnavailable: 0` (из §2.1) вывод `kubectl get endpoints web` ни в какой момент не должен опускаться ниже 5. **Проверь это во втором терминале.**

### 4.5 — Proof of work

4.5 — Доказательства работы

**Paste into `docs/LAB09.md`:**

**Вставь в `docs/LAB09.md`:**

- The **kube-DNS proof** — your `kubectl exec $WEBPOD -- ... http://echo/ping` command and its `pong` output. **This is the headline artifact.**
  * **Доказательство kube-DNS** — команда `kubectl exec $WEBPOD -- ... http://echo/ping` и её вывод `pong`. **Это главный артефакт.**
- The load-balancing capture — 6+ hostnames showing **both** echo pod names appearing roughly evenly
  * Скриншот балансировки — 6+ имён хостов, где **оба** пода echo встречаются примерно поровну
- `kubectl get pods -l app=web -o wide` after scaling to 5 — 5 pods, spread across nodes
  * `kubectl get pods -l app=web -o wide` после масштабирования до 5 — пять подов, распределённых по нодам
- `kubectl rollout history deploy/web` — at least 2 revisions visible
  * `kubectl rollout history deploy/web` — видно как минимум 2 ревизии
- `kubectl rollout undo` output + a `kubectl rollout status` confirming the rollback succeeded
  * Вывод `kubectl rollout undo` и `kubectl rollout status`, подтверждающий успешный откат
- 2–3 sentences on the most useful `kubectl describe` / `kubectl get events` finding you hit while debugging — what was the actual `Events:` line that told you the answer?
  * 2–3 предложения о самой полезной находке из `kubectl describe` / `kubectl get events` при отладке — какая именно строка `Events:` дала ответ?

---

## Task 5 — Documentation (1 pt)

Задача 5 — документация (1 балл)

`YOUR TASK`: write `k8s/docs/LAB09.md` with these sections, in order:

`YOUR TASK`: напиши `k8s/docs/LAB09.md` со следующими разделами по порядку:

1. **Architecture** — a Mermaid diagram showing `User → web Service → 3× web pods` and `web pod → echo Service → 2× echo pods`, with the kube-DNS resolution marked on the second arrow
   * **Архитектура** — схема Mermaid с `User → web Service → 3× web pods` и `web pod → echo Service → 2× echo pods`, где на второй стрелке отмечено разрешение имени в kube-DNS
2. **Cluster** — your `k3d cluster create` command + the 3-question research from §1.1
   * **Кластер** — твоя команда `k3d cluster create` и ответы на три вопроса из §1.1
3. **Manifests** — one short paragraph per file: which labels, which Service type, which replica count, which probe endpoints — and **why** for each non-obvious choice
   * **Манифесты** — по короткому абзацу на файл: какие метки, какой тип Service, сколько реплик, какие эндпоинты в пробах — и **почему** для каждого неочевидного выбора
4. **Cross-service evidence** — the §4.1 `pong` capture, with a sentence explaining each of the six things that just worked
   * **Подтверждение межсервисного вызова** — скриншот с `pong` из §4.1 и по предложению на каждую из шести сработавших вещей
5. **Operations** — scaling, rolling update, rollback evidence from §4.3–4.4
   * **Операции** — доказательства масштабирования, rolling update и отката из §4.3–4.4
6. **Challenges & solutions** — at least one real one (not "I was new to YAML"); include the `kubectl describe` line that unblocked you
   * **Трудности и решения** — хотя бы одна реальная (не «я новичок в YAML»); приведи строку из `kubectl describe`, которая помогла разобраться

Include manifest **snippets** (not whole files — those are committed already) and the real CLI captures from Tasks 1–4.

Вставляй **фрагменты** манифестов (не целиком — файлы уже в репозитории) и настоящие выводы команд из задач 1–4.

---

## Bonus Task — Ingress with TLS through Traefik (2 pts)

Бонусная задача — Ingress с TLS через Traefik (2 балла)

Less hand-holding. k3d ships Traefik built in (it's the LoadBalancer behind the `:8080`/`:8443` host ports you mapped in §1.2) — **no controller install**. Use `*.localhost` (resolves to `127.0.0.1` automatically — no `/etc/hosts` edits).

Здесь подсказок меньше. В k3d Traefik уже встроен (это LoadBalancer за портами хоста `:8080` и `:8443`, которые ты пробросил в §1.2) — **контроллер ставить не надо**. Используй `*.localhost` (сам резолвится в `127.0.0.1` — править `/etc/hosts` не нужно).

`YOUR TASK`: route `web.localhost` → `web` Service and `echo.localhost` → `echo` Service, both over HTTPS, through a single `Ingress`.

`YOUR TASK`: направь `web.localhost` → Service `web`, а `echo.localhost` → Service `echo`, оба по HTTPS, через один `Ingress`.

1. **Self-signed cert + Secret** covering both hostnames:
   * **Самоподписанный сертификат + Secret** для обоих имён хостов:
   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout tls.key -out tls.crt \
     -subj "/CN=web.localhost" \
     -addext "subjectAltName=DNS:web.localhost,DNS:echo.localhost"
   kubectl create secret tls ___ --key tls.key --cert tls.crt   # YOUR TASK: pick a Secret name
   ```

Пояснение: `openssl` создаёт самоподписанный сертификат с обоими именами в SAN, затем `kubectl create secret tls` кладёт ключ и сертификат в Secret.

2. **`k8s/ingress.yaml`** — Ingress shape:
   * **`k8s/ingress.yaml`** — структура Ingress:

```yaml
apiVersion: ___                       # YOUR TASK: which group/version owns Ingress? (NOT extensions/v1beta1 — that's gone)
kind: ___
metadata:
  name: ___
spec:
  ingressClassName: ___               # YOUR TASK: k3d's built-in controller name (lecture 9 slide 13)
  tls:
    - hosts: [___, ___]               # YOUR TASK: both hostnames
      secretName: ___                 # YOUR TASK: matches the Secret name above
  rules:
    - host: ___                       # YOUR TASK: web's hostname
      http:
        paths:
          - path: ___                 # YOUR TASK
            pathType: ___             # YOUR TASK: Prefix or Exact?
            backend:
              service:
                name: ___             # YOUR TASK: which Service is this rule routing to?
                port:
                  number: ___         # YOUR TASK: the Service's port (NOT the targetPort).
                                      #            Or use `name: http` to reference the named Service port.
    - host: ___                       # YOUR TASK: echo's hostname
      http:
        paths:
          - path: ___
            pathType: ___
            backend:
              service:
                name: ___
                port:
                  number: ___         # (same options as above — number or name)
```

Пояснение: в `tls` перечисляются оба хоста и имя Secret, а в `rules` для каждого хоста задаётся путь и Service-бэкенд.

3. **Verify** via the host port you mapped at cluster create:
   * **Проверка** через порт хоста, проброшенный при создании кластера:
   ```bash
   curl -k https://echo.localhost:8443/ping    # → pong, through Traefik + TLS
   curl -k https://web.localhost:8443/health   # → your app's health JSON
   ```

Пояснение: `-k` отключает проверку самоподписанного сертификата; запросы идут через Traefik на порт 8443 хоста.

> 💡 **Host-based routing** (one Service per hostname) avoids path-rewrite middleware, so each backend sees the request path unchanged. The alternative — path-based (`/web` → web, `/echo` → echo on a single hostname) — needs Traefik middleware to strip the prefix; not worth the complication for two services.
>
> 💡 **Маршрутизация по хосту** (один Service на одно имя хоста) избавляет от middleware для перезаписи пути, поэтому каждый бэкенд видит путь запроса без изменений. Альтернатива — маршрутизация по пути (`/web` → web, `/echo` → echo на одном хосте) — требует middleware Traefik, срезающего префикс; ради двух сервисов это лишнее усложнение.

**Evidence (paste into `docs/LAB09.md`):**

**Доказательства (вставь в `docs/LAB09.md`):**

- `kubectl get pods -n kube-system | grep traefik` — controller Running
  * `kubectl get pods -n kube-system | grep traefik` — контроллер в состоянии Running
- `kubectl get ingress` — your Ingress with the two hosts visible
  * `kubectl get ingress` — твой Ingress с двумя видимыми хостами
- Both `curl -k https://...localhost:8443/...` captures, real output
  * Оба вывода `curl -k https://...localhost:8443/...` с реальным результатом
- 1–2 sentences on what Ingress buys you over the NodePort Service from §2.2
  * 1–2 предложения о том, что даёт Ingress по сравнению с NodePort Service из §2.2

---

## How to Submit

Как сдать

```bash
git switch -c lab09
git add k8s/
git commit -m "feat(lab09): web + echo on k3d — cross-service kube-DNS proven"
git push -u origin lab09
```

Пояснение: создай ветку `lab09`, добавь каталог `k8s/`, закоммить и запушь ветку в свой форк.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab09` → `course-repo:master` *(reviewed)*
  * `your-fork:lab09` → `course-repo:master` *(с ревью)*
- `your-fork:lab09` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab09` → `your-fork:master` *(вливается в твой main по завершении)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — 3-node k3d cluster on K8s 1.36.1, evidence captured
- [ ] Task 2 done — web Deployment (3 replicas) + Service, label triangle correct, probes on cheap endpoint
- [ ] Task 3 done — echo Deployment (2 replicas) + ClusterIP Service, sized for a tiny Go binary
- [ ] Task 4 done — kubectl exec from inside web pod returns `pong` from echo; LB rotation shown; scaled to 5; rolled out & back
- [ ] Task 5 done — LAB09.md with all 6 sections + Mermaid diagram + the pong capture
- [ ] Bonus done — Ingress over HTTPS via Traefik, both hostnames reachable
```

Пояснение: это чек-лист для описания PR — отметь выполненные пункты.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — k3d cluster (2 pts)

Задача 1 — кластер k3d (2 балла)

- ✅ `kubectl version` reports server `1.36.x`
  * ✅ `kubectl version` показывает сервер `1.36.x`
- ✅ `kubectl get nodes -o wide` shows **3 nodes**, all `Ready`
  * ✅ `kubectl get nodes -o wide` показывает **3 ноды**, все `Ready`
- ✅ LoadBalancer host-port maps `:80→:8080` and `:443→:8443` present in the `k3d cluster create` command
  * ✅ В команде `k3d cluster create` есть проброс портов LoadBalancer `:80→:8080` и `:443→:8443`
- ✅ Research answers from §1.1 in `docs/LAB09.md`
  * ✅ Ответы на вопросы из §1.1 в `docs/LAB09.md`

### Task 2 — web service (3 pts)

Задача 2 — сервис web (3 балла)

- ✅ `web-deployment.yaml` and `web-service.yaml` written by hand
  * ✅ `web-deployment.yaml` и `web-service.yaml` написаны вручную
- ✅ Label triangle consistent (Deployment labels = selector.matchLabels = template labels)
  * ✅ Треугольник меток согласован (метки Deployment = selector.matchLabels = метки шаблона)
- ✅ **Container port AND Service port both named** (e.g. `name: http`) — Lab 10/16 require this
  * ✅ **Имя задано и для порта контейнера, и для порта Service** (например, `name: http`) — это требуется в лабах 10 и 16
- ✅ 3 replicas Ready; `kubectl get endpoints web` shows **3 IPs**
  * ✅ 3 реплики в состоянии Ready; `kubectl get endpoints web` показывает **3 IP**
- ✅ Resource **requests AND limits** set; both probes wired to a `/health`-style endpoint, not the app root
  * ✅ Заданы и **requests, и limits**; обе пробы смотрят на эндпоинт вида `/health`, а не на корень приложения
- ✅ Pods spread across all 3 nodes (`-o wide` capture)
  * ✅ Поды распределены по всем трём нодам (скриншот с `-o wide`)

### Task 3 — echo service (2 pts)

Задача 3 — сервис echo (2 балла)

- ✅ `echo-deployment.yaml` uses `ghcr.io/inno-devops-labs/echo:v1` (no self-builds)
  * ✅ `echo-deployment.yaml` использует `ghcr.io/inno-devops-labs/echo:v1` (без собственной сборки)
- ✅ **2** replicas Ready; ClusterIP Service has **2** endpoints
  * ✅ **2** реплики Ready; у ClusterIP Service **2** эндпоинта
- ✅ Probes wired to `/healthz:8081` (echo's actual readiness endpoint, per `plumbing/echo/README.md`)
  * ✅ Пробы смотрят на `/healthz:8081` (настоящий эндпоинт готовности echo, см. `plumbing/echo/README.md`)
- ✅ Resources sized smaller than web (tiny Go binary)
  * ✅ Ресурсы меньше, чем у web (крошечный Go-бинарник)

### Task 4 — cross-service + ops (2 pts)

Задача 4 — межсервисное взаимодействие и операции (2 балла)

- ✅ **`kubectl exec` into a web pod + call to `http://echo/ping` returns `pong`** — kube-DNS proof captured
  * ✅ **`kubectl exec` в под web и вызов `http://echo/ping` возвращает `pong`** — доказательство kube-DNS сохранено
- ✅ Load-balancing capture shows **both** echo pod hostnames appearing
  * ✅ На скриншоте балансировки видны **оба** имени подов echo
- ✅ `web` scaled to 5; endpoints list reaches 5; no downtime visible
  * ✅ `web` масштабирован до 5; в списке endpoints пять IP; простоя не видно
- ✅ Rolling update + rollback shown with `kubectl rollout history` evidence
  * ✅ Показаны rolling update и откат с подтверждением через `kubectl rollout history`

### Task 5 — docs (1 pt)

Задача 5 — документация (1 балл)

- ✅ All 6 sections in `k8s/docs/LAB09.md`; Mermaid diagram included; the pong capture is the headline artefact
  * ✅ Все 6 разделов в `k8s/docs/LAB09.md`; есть схема Mermaid; скриншот с pong — главный артефакт

### Bonus — Ingress + TLS (2 pts)

Бонус — Ingress + TLS (2 балла)

- ✅ Self-signed Secret created with both SANs
  * ✅ Создан Secret с самоподписанным сертификатом с обоими SAN
- ✅ Ingress uses `ingressClassName: traefik`, both hosts routed
  * ✅ В Ingress указан `ingressClassName: traefik`, оба хоста маршрутизируются
- ✅ Both `curl -k https://...localhost:8443/...` calls succeed
  * ✅ Оба вызова `curl -k https://...localhost:8443/...` успешны

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — k3d cluster | **2** | 3-node K8s 1.36 cluster up, research answers, port maps for bonus |
| **Task 2** — web service | **3** | Hand-written Deployment + Service, label triangle correct, probes on cheap endpoint, requests+limits, 3 endpoints |
| **Task 3** — echo service | **2** | Provided image deployed (2 replicas), ClusterIP Service, probes on `/healthz` |
| **Task 4** — cross-service + ops | **2** | `pong` from inside web pod, LB rotation, scale+rollout+rollback |
| **Task 5** — docs | **1** | 6 sections, Mermaid, pong capture as headline |
| **Bonus** — Ingress + TLS | **2** | Traefik Ingress, both hostnames over HTTPS |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — кластер k3d | **2** | Поднят 3-нодовый кластер K8s 1.36, есть ответы на вопросы, проброшены порты для бонуса |
| **Задача 2** — сервис web | **3** | Написанные вручную Deployment + Service, верный треугольник меток, пробы на дешёвом эндпоинте, requests+limits, 3 эндпоинта |
| **Задача 3** — сервис echo | **2** | Развёрнут предоставленный образ (2 реплики), ClusterIP Service, пробы на `/healthz` |
| **Задача 4** — межсервисное взаимодействие и операции | **2** | `pong` изнутри пода web, ротация при балансировке, масштабирование + rollout + откат |
| **Задача 5** — документация | **1** | 6 разделов, Mermaid, pong-скриншот как главный артефакт |
| **Бонус** — Ingress + TLS | **2** | Ingress Traefik, оба хоста по HTTPS |
| **Итого** | **12** | 10 основных + 2 бонусных |

---

## Resources

Ресурсы

<details>
<summary>📚 Kubernetes documentation</summary>

- [Kubernetes 1.36 "Haru" release notes](https://kubernetes.io/blog/2026/04/22/kubernetes-v1-36-release/)
  * [Release notes Kubernetes 1.36 «Haru»](https://kubernetes.io/blog/2026/04/22/kubernetes-v1-36-release/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
  * [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services & networking](https://kubernetes.io/docs/concepts/services-networking/service/)
  * [Services и сеть](https://kubernetes.io/docs/concepts/services-networking/service/)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
  * [DNS для сервисов и подов](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Liveness, readiness, startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
  * [Пробы liveness, readiness и startup](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
  * [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [kubectl cheat sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
  * [Шпаргалка по kubectl](https://kubernetes.io/docs/reference/kubectl/quick-reference/)

</details>

<details>
<summary>🛠️ Tools</summary>

- [k3d](https://k3d.io/) — k3s in Docker
  * [k3d](https://k3d.io/) — k3s в Docker
- [k3s docs](https://docs.k3s.io/)
  * [Документация k3s](https://docs.k3s.io/)
- [k9s](https://k9scli.io/) — TUI for kubectl
  * [k9s](https://k9scli.io/) — TUI для kubectl
- [kubectx/kubens](https://github.com/ahmetb/kubectx) — context + namespace switcher
  * [kubectx/kubens](https://github.com/ahmetb/kubectx) — переключение контекстов и пространств имён
- [stern](https://github.com/stern/stern) — multi-pod log tailing
  * [stern](https://github.com/stern/stern) — просмотр логов нескольких подов одновременно

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

- **Label-selector mismatch — the #1 K8s YAML error.** The Deployment has *three* places where labels appear: `metadata.labels`, `spec.selector.matchLabels`, and `spec.template.metadata.labels`. The Service's `spec.selector` is a fourth. They all need to agree. Symptom: `kubectl get endpoints <svc>` is empty even though pods are Running. The selector is **immutable** after Deployment creation — if you got it wrong, you must `kubectl delete deploy/...` and re-apply (you can't `kubectl edit` your way out).
  * **Несовпадение меток и селектора — ошибка YAML для K8s номер один.** В Deployment есть *три* места с метками: `metadata.labels`, `spec.selector.matchLabels` и `spec.template.metadata.labels`. `spec.selector` у Service — четвёртое. Везде должны быть согласованные значения. Симптом: `kubectl get endpoints <svc>` пуст, хотя поды в состоянии Running. Селектор **неизменяем** после создания Deployment — если ошибся, нужно `kubectl delete deploy/...` и применить заново (`kubectl edit` тут не спасёт).

- **`imagePullPolicy: Always` fails for locally-built images on k3d.** `k3d image import` populates the cluster's containerd directly; there's no registry holding your `:latest` tag. `Always` re-pulls every restart and gets `ErrImagePull` / `ImagePullBackOff`. Use `IfNotPresent` for locally-imported images (this is also the default when your tag is anything other than `:latest` — but **don't rely on the default**; declare it).
  * **`imagePullPolicy: Always` не работает для локально собранных образов на k3d.** `k3d image import` кладёт образ прямо в containerd кластера; реестра, где лежит твой тег `:latest`, нет. `Always` при каждом перезапуске пытается скачать образ заново и получает `ErrImagePull` / `ImagePullBackOff`. Для локально импортированных образов используй `IfNotPresent` (это также значение по умолчанию, если тег не `:latest`, но **не полагайся на умолчание** — указывай явно).

- **NodePort on k3d needs the host port mapped at cluster create.** k3d's klipper-lb exposes the LoadBalancer-class ports you listed in `-p`, but a NodePort in the 30000–32767 range isn't reachable from your laptop unless you either (a) added `-p "3xxxx:3xxxx@server:0"` at cluster create, or (b) `kubectl port-forward`. `LoadBalancer` Services and `port-forward` both work without ceremony; raw NodePort doesn't.
  * **NodePort в k3d требует проброса порта хоста при создании кластера.** klipper-lb в k3d открывает порты класса LoadBalancer, которые ты указал в `-p`, но NodePort из диапазона 30000–32767 недоступен с ноутбука, если ты (а) не добавил `-p "3xxxx:3xxxx@server:0"` при создании кластера или (б) не сделал `kubectl port-forward`. Сервисы `LoadBalancer` и `port-forward` работают без лишних телодвижений, а «голый» NodePort — нет.

- **`livenessProbe` pointing at the app's main page.** When traffic spikes, your handler slows. Liveness times out, K8s restarts the container, the queue drains onto the remaining pods, *they* slow under doubled load — **cascading restarts** during the exact moment you needed the system to stay up. Liveness must point at a **cheap, in-process** signal (`/health` returning 200 with no DB calls). That's why Lab 1 made you build `/health` separately.
  * **`livenessProbe`, смотрящая на главную страницу приложения.** При всплеске трафика обработчик замедляется. Проба liveness завершается по таймауту, K8s перезапускает контейнер, очередь перетекает на оставшиеся поды, *они* замедляются под двойной нагрузкой — **каскадные перезапуски** именно в тот момент, когда системе нужно было оставаться на ногах. Liveness должна смотреть на **дешёвый, локальный** сигнал (`/health`, возвращающий 200 без обращений к БД). Именно поэтому в лабе 1 нужно было сделать `/health` отдельно.

- **Missing or slow `readinessProbe` causes rolling-update gaps.** Without readiness, K8s adds a pod to the Service endpoints the instant the container *starts* — before your app is actually serving. During a rolling update, that means a few seconds of `connection refused` on every replacement. Readiness with `periodSeconds: 3–5` and a low `failureThreshold` keeps unready pods out of the endpoint list and out of traffic.
  * **Отсутствие или медленная `readinessProbe` создаёт провалы при rolling update.** Без readiness K8s добавляет под в endpoints Service в момент, когда контейнер только *стартовал*, — то есть до того, как приложение реально готово отвечать. Во время rolling update это даёт несколько секунд `connection refused` на каждой замене. Readiness с `periodSeconds: 3–5` и низким `failureThreshold` держит неготовые поды вне списка endpoints и вне трафика.

- **`port` vs `targetPort` confusion.** `port` is what *clients* hit on the Service; `targetPort` is what the **container** listens on. Common error: `port: 8081` on the echo Service. It works — but every caller now writes `http://echo:8081/ping` and your future Helm chart, Lab 13 ApplicationSet, and Lab 16 ServiceMonitor all have to special-case the port. Pick `port: 80` for HTTP services so DNS-only URLs work.
  * **Путаница `port` и `targetPort`.** `port` — это то, куда обращаются *клиенты* через Service; `targetPort` — то, что слушает **контейнер**. Частая ошибка: `port: 8081` у Service echo. Работает, но теперь каждый вызывающий пишет `http://echo:8081/ping`, а твой будущий Helm-чарт, ApplicationSet в лабе 13 и ServiceMonitor в лабе 16 вынуждены особым образом обрабатывать этот порт. Для HTTP-сервисов выбирай `port: 80`, чтобы URL без порта работал.

- **Resource requests too low → noisy-neighbour starvation.** k3d on a laptop has finite memory. If `web`'s `requests: 64Mi` is realistic but you set `8Mi`, the scheduler over-packs nodes and your pods get OOM-evicted under any real workload. Set `requests` to what your app actually uses at idle + a small buffer, not the minimum it can boot with.
  * **Слишком низкие requests → голодание из-за «шумных соседей».** Память в k3d на ноутбуке ограничена. Если `requests: 64Mi` для `web` реалистичны, а ты поставил `8Mi`, планировщик переполнит ноды, и при любой реальной нагрузке поды будут вытеснены по OOM. Ставь в `requests` реальное потребление приложения в простое плюс небольшой запас, а не минимум, с которым оно может загрузиться.

- **`kubectl edit` instead of `kubectl apply -f`.** `edit` mutates the live object directly; your manifest in git no longer matches reality. Next `apply` reverts your hotfix. In a GitOps world (Lab 13+) this is a deploy-time race condition. Always edit the YAML, `apply -f`, commit.
  * **`kubectl edit` вместо `kubectl apply -f`.** `edit` меняет живой объект напрямую, и твой манифест в git перестаёт соответствовать реальности. Следующий `apply` откатит твой хотфикс. В мире GitOps (лаба 13 и далее) это гоночная ситуация на этапе деплоя. Всегда правь YAML, делай `apply -f` и коммить.

- **Container missing `curl`/`wget` and you reach for the wrong workaround.** Don't `port-forward` to "test" cross-service calls — that proves your laptop reaches echo, not that `web` does. Use `kubectl run tmp --rm -it --image=curlimages/curl ...` or `kubectl exec $WEBPOD -- python -c "..."` so the call originates **inside the cluster**.
  * **В контейнере нет `curl`/`wget`, и ты хватаешься за неправильный обходной путь.** Не делай `port-forward` для «проверки» межсервисных вызовов — это доказывает, что с ноутбука доступен echo, а не что его видит `web`. Используй `kubectl run tmp --rm -it --image=curlimages/curl ...` или `kubectl exec $WEBPOD -- python -c "..."`, чтобы запрос исходил **изнутри кластера**.

- **k3d's bundled Traefik vs ingress-nginx muscle memory.** Tutorials online assume ingress-nginx (`ingressClassName: nginx`, `nginx.ingress.kubernetes.io/*` annotations). On k3d it's Traefik (`ingressClassName: traefik`, `traefik.ingress.kubernetes.io/*` annotations). The Kubernetes `Ingress` core fields are portable; annotations are not.
  * **Встроенный Traefik в k3d против привычки к ingress-nginx.** Онлайн-уроки обычно рассчитаны на ingress-nginx (`ingressClassName: nginx`, аннотации `nginx.ingress.kubernetes.io/*`). В k3d это Traefik (`ingressClassName: traefik`, аннотации `traefik.ingress.kubernetes.io/*`). Базовые поля `Ingress` в Kubernetes переносимы, а аннотации — нет.

</details>

<details>
<summary>🔍 Debugging cheatsheet</summary>

```bash
kubectl describe pod <name>           # bottom Events section = root-cause oracle
kubectl get events --sort-by=.lastTimestamp  # cluster-wide event stream
kubectl logs <pod> --previous         # logs from the previous (crashed) container
kubectl get endpoints <svc>           # empty = selector/label mismatch
kubectl exec -it <pod> -- /bin/sh     # shell into the pod (if it has one)
kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -- bash
                                       # DNS + network sandbox with dig, nslookup, curl, tcpdump
```

Пояснение: это первая линия диагностики — пустой список endpoints означает ошибку селектора, а `--previous` показывает логи упавшего контейнера.

</details>

---

## Looking Ahead

Взгляд вперёд

| Lab | What it adds to this stack |
|---:|---|
| 10 | Re-package these manifests as a **Helm 4** chart; `echo` becomes a subchart |
| 11 | Replace plaintext config with **Secrets** (and meet OpenBao) |
| 12 | **ConfigMaps** + **PVCs** — config injection + state survival across pod restarts |
| 13 | **ArgoCD** GitOps — `web` + `echo` deployed via an `ApplicationSet` for 2 envs |
| 14 | **Argo Rollouts** — canary the `echo` Deployment with progressive traffic shift |
| 16 | **kube-prometheus-stack** — scrape `echo`'s `/metrics` via a `ServiceMonitor` |

| Лаба | Что добавляет к этому стеку |
|---:|---|
| 10 | Переупаковка этих манифестов в чарт **Helm 4**; `echo` становится сабчартом |
| 11 | Замена конфигурации в открытом виде на **Secrets** (и знакомство с OpenBao) |
| 12 | **ConfigMaps** + **PVCs** — инъекция конфигурации и сохранение состояния между перезапусками подов |
| 13 | **ArgoCD** GitOps — `web` и `echo` разворачиваются через `ApplicationSet` для двух окружений |
| 14 | **Argo Rollouts** — канареечное развёртывание `echo` Deployment с постепенным переносом трафика |
| 16 | **kube-prometheus-stack** — сбор `/metrics` сервиса `echo` через `ServiceMonitor` |

---

**Good luck!** ☸️

**Удачи!** ☸️

> **Remember:** the K8s networking model only clicks once you have a **second** service to talk to. That `pong` returned by `echo` — to a `curl` run *inside* your `web` pod — is the moment everything in lecture 9 (Pod, Service, label selector, kube-DNS, endpoints, kube-proxy) lines up. If only one capture survives the lab, make it that one.
>
> **Запомни:** сетевая модель K8s осмысляется только тогда, когда есть **второй** сервис, с которым можно поговорить. Тот самый `pong` от `echo` в ответ на `curl`, запущенный *внутри* пода `web`, — момент, когда всё из лекции 9 (Pod, Service, селектор меток, kube-DNS, endpoints, kube-proxy) складывается в единую картину. Если из лабы должен остаться только один скриншот, пусть это будет он.
