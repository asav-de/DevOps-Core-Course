# 📌 Lecture 13 — GitOps with ArgoCD: Git as the Source of Truth

Лекция 13 — GitOps с ArgoCD: Git как источник истины

## 📍 Slide 1 – 🚀 Welcome to GitOps

Слайд 1 — Добро пожаловать в GitOps

* 🌍 **Lectures 9-12 left you with a real cluster** running a Helm-packaged Python service plus a Go echo sidecar, with secrets in OpenBao and persistent state on PVCs. Deployment is still you typing `helm upgrade` from a laptop.
  * 🌍 **Лекции 9–12 оставили тебе настоящий кластер** с Python-сервисом, упакованным в Helm, и сайдкаром Go echo; секреты лежат в OpenBao, состояние — на PVC. А деплой по-прежнему — это ты вручную набираешь `helm upgrade` с ноутбука.
* 🤖 **GitOps** moves that "laptop" *inside* the cluster: an agent watches Git, pulls the desired state, and reconciles continuously. No more out-of-band `kubectl` from human hands.
  * 🤖 **GitOps** переносит этот «ноутбук» *внутрь* кластера: агент следит за Git, забирает желаемое состояние и непрерывно приводит кластер к нему. Никаких ручных `kubectl` в обход процесса.
* 🎯 This lecture: the GitOps mental model + **ArgoCD 3.4** (May 2026) architecture, Application + ApplicationSet CRDs, sync policies, the App-of-Apps pattern, and why this only becomes interesting once you have **three** services.
  * 🎯 В этой лекции: ментальная модель GitOps + архитектура **ArgoCD 3.4** (май 2026), CRD `Application` и `ApplicationSet`, политики синхронизации, паттерн App-of-Apps — и почему всё это становится интересным только когда у тебя **три** сервиса.
* 🔗 **Tie-in to Lab 13:** install ArgoCD via Helm, declare Application manifests for `app-python`, `app-go-echo`, **and a third `app-go-health` plumbing service**, and let an `ApplicationSet` generate dev + prod copies from one template.
  * 🔗 **Связь с лабой 13:** установить ArgoCD через Helm, объявить манифесты Application для `app-python`, `app-go-echo` **и третьего вспомогательного сервиса `app-go-health`**, а генерацию dev- и prod-копий отдать одному `ApplicationSet`.

```mermaid
flowchart LR
  Dev[👨‍💻 Developer] -->|git push| Repo[📝 Git]
  Repo -->|pull every 3m| Argo[🤖 ArgoCD in cluster]
  Argo -->|reconcile| K8s[☸️ Kubernetes]
  K8s -->|status| Argo
```

Схема: 👨‍💻 разработчик делает git push в 📝 Git, 🤖 ArgoCD в кластере опрашивает репозиторий каждые 3 минуты, применяет изменения к ☸️ Kubernetes и получает обратно статус.

---

## 📍 Slide 2 – 🎯 Learning Outcomes

Слайд 2 — Результаты обучения

| # | Outcome |
|---|---------|
| 1 | 📜 Name the **four OpenGitOps principles** and explain why each matters |
| 2 | 🔁 Contrast **push vs pull** deployment models on credential surface, drift, audit |
| 3 | 🏗️ Sketch the **ArgoCD 3.4 architecture**: argocd-server, application-controller, repo-server, redis, Dex |
| 4 | 📝 Read & write an **Application** CRD with `source`, `destination`, `syncPolicy` |
| 5 | 🌳 Compose deployments with the **App-of-Apps pattern** and **ApplicationSet** generators |
| 6 | 🛡️ Reason about **sync waves, hooks, self-heal, prune**, and when manual sync wins |

| # | Результат |
|---|---------|
| 1 | 📜 Назвать **четыре принципа OpenGitOps** и объяснить, почему каждый важен |
| 2 | 🔁 Сравнить модели доставки **push и pull** по поверхности учётных данных, дрейфу и аудиту |
| 3 | 🏗️ Нарисовать **архитектуру ArgoCD 3.4**: argocd-server, application-controller, repo-server, redis, Dex |
| 4 | 📝 Читать и писать **Application** CRD с полями `source`, `destination`, `syncPolicy` |
| 5 | 🌳 Собирать деплои через **паттерн App-of-Apps** и генераторы **ApplicationSet** |
| 6 | 🛡️ Рассуждать о **sync waves, hooks, self-heal, prune** и о том, когда ручная синхронизация лучше |

**Tech stack pinned for May 2026:** **ArgoCD 3.4.x** (GA early May 2026; **2.x is EOL since November 2025 with the 3.2 release**), Kubernetes **1.36** "Haru", a single GitOps repo on GitHub, deployments driven from your Lab 10 Helm chart.

**Стек технологий, зафиксированный на май 2026:** **ArgoCD 3.4.x** (GA в начале мая 2026; **ветка 2.x EOL с ноября 2025 вместе с релизом 3.2**), Kubernetes **1.36** «Haru», один GitOps-репозиторий на GitHub, деплой из твоего Helm-чарта из лабы 10.

---

## 📍 Slide 3 – ❓ Why Pull Beats Push

Слайд 3 — Почему pull лучше push

You shipped Lab 12 with a CI workflow that runs `helm upgrade` against the cluster. It works. So why move?

В лабе 12 ты сдал CI-процесс, который выполняет `helm upgrade` на кластере. Он работает. Зачем же что-то менять?

* 🔑 **Credentials.** CI needs a kubeconfig with cluster-admin to apply. That kubeconfig lives in GitHub secrets, on developer laptops, in scripts. Every leak is a cluster compromise.
  * 🔑 **Учётные данные.** Для применения CI нужен kubeconfig с правами cluster-admin. Этот kubeconfig лежит в секретах GitHub, на ноутбуках разработчиков, в скриптах. Каждая утечка — это компрометация кластера.
* 🌊 **Drift.** Someone `kubectl edit`s a Deployment at 3am to "fix" a prod incident. The Helm chart in Git no longer matches reality. Next `helm upgrade` silently overwrites the fix.
  * 🌊 **Дрейф.** Кто-то в 3 часа ночи делает `kubectl edit` Deployment, чтобы «починить» продовый инцидент. Helm-чарт в Git больше не отражает реальность. Следующий `helm upgrade` молча перезапишет этот фикс.
* 🕵️ **Audit.** "Who scaled the prod ingress to 12 replicas?" — the answer lives in shell history, not Git.
  * 🕵️ **Аудит.** «Кто масштабировал продовый ingress до 12 реплик?» — ответ живёт в истории shell, а не в Git.
* 🚑 **Recovery.** Cluster gone? With push CI/CD you re-run pipelines; with GitOps you `helm install argo-cd && kubectl apply -f root.yaml` and the cluster rebuilds itself.
  * 🚑 **Восстановление.** Кластера больше нет? С push CI/CD ты перезапускаешь пайплайны; с GitOps ты выполняешь `helm install argo-cd && kubectl apply -f root.yaml`, и кластер пересобирает себя сам.

> 🔥 **Hot take:** GitOps is not a tool, it's an *inversion of control*. The cluster reaches out to Git; nothing reaches into the cluster.
>
> 🔥 **Резко, но честно:** GitOps — не инструмент, а *инверсия управления*. Кластер сам тянется к Git; ничто не тянется внутрь кластера.

---

## 📍 Slide 4 – 📜 The Four OpenGitOps Principles

Слайд 4 — Четыре принципа OpenGitOps

The CNCF **OpenGitOps** working group ratified four principles (v1.0.0). Memorize them — every GitOps tool implements these or it isn't GitOps.

Рабочая группа CNCF **OpenGitOps** утвердила четыре принципа (v1.0.0). Выучи их — каждый GitOps-инструмент реализует именно их, иначе это не GitOps.

1. 📝 **Declarative** — the desired state is described in declarative form (YAML, HCL, …), never imperative commands.
   * 📝 **Декларативность** — желаемое состояние описано в декларативной форме (YAML, HCL, …), а не императивными командами.
2. 🔒 **Versioned and Immutable** — that desired state is stored with full history; you can always check out commit `abc123` and know what was deployed.
   * 🔒 **Версионирование и неизменяемость** — желаемое состояние хранится с полной историей; ты всегда можешь переключиться на коммит `abc123` и узнать, что было развёрнуто.
3. 🤖 **Pulled Automatically** — software agents *inside* the target system pull the desired state. Nobody pushes from outside.
   * 🤖 **Автоматическое вытягивание** — программные агенты *внутри* целевой системы сами забирают желаемое состояние. Никто не проталкивает его снаружи.
4. ♾️ **Continuously Reconciled** — agents compare actual to desired and automatically act to converge them. Forever.
   * ♾️ **Непрерывное согласование** — агенты сравнивают фактическое состояние с желаемым и автоматически сводят их вместе. Бесконечно.

> 💡 **Litmus test:** if you removed the agent, would the cluster drift from Git? If yes, you have GitOps. If no (because nothing was reconciling), you just had "YAML in Git".
>
> 💡 **Лакмусовая бумажка:** если убрать агента, кластер разойдётся с Git? Если да — у тебя GitOps. Если нет (потому что ничего не согласовывало), у тебя был просто «YAML в Git».

---

## 📍 Slide 5 – 🔄 Push vs Pull, Side by Side

Слайд 5 — Push и pull бок о бок

```mermaid
flowchart LR
  subgraph Push["🔄 Push Model (classic CI/CD)"]
    direction LR
    G1[📝 Git] --> CI1[🔧 CI runner]
    CI1 -->|kubeconfig + apply| K1[☸️ Cluster]
  end
  subgraph Pull["🚀 Pull Model (GitOps)"]
    direction LR
    G2[📝 Git] -.->|polled / webhook| A2[🤖 Agent in cluster]
    A2 -->|apply locally| K2[☸️ Cluster]
  end
```

Схема: в push-модели CI-раннер с kubeconfig применяет манифесты в кластер; в pull-модели агент внутри кластера сам опрашивает Git (или получает webhook) и применяет изменения локально.

| Aspect | 🔄 Push CI/CD | 🚀 Pull GitOps |
|--------|--------------|----------------|
| Cluster credentials | Live in CI secrets | Stay inside the cluster |
| Drift detection | None — CI runs and forgets | Continuous, every 3 minutes |
| Audit trail | CI logs (rotated) | `git log` (immutable) |
| Disaster recovery | Re-run pipeline against new cluster | Install agent, point at Git |
| What blocks deploy | CI runner availability | Git availability |

| Аспект | 🔄 Push CI/CD | 🚀 Pull GitOps |
|--------|--------------|----------------|
| Учётные данные кластера | Живут в секретах CI | Остаются внутри кластера |
| Обнаружение дрейфа | Нет — CI отработал и забыл | Непрерывно, каждые 3 минуты |
| Аудит | Логи CI (ротируются) | `git log` (неизменяемый) |
| Восстановление после катастрофы | Перезапустить пайплайн на новом кластере | Установить агент, указать на Git |
| Что блокирует деплой | Доступность CI-раннера | Доступность Git |

---

## 📍 Slide 6 – 🛠️ The GitOps Tools Landscape

Слайд 6 — Ландшафт GitOps-инструментов

| Tool | Lineage | Sweet spot |
|------|---------|-----------|
| **ArgoCD** | CNCF graduated (Intuit, 2018) | UI-first, declarative `Application` CRD, sync waves |
| **Flux CD v2** | CNCF graduated (Weaveworks, 2020) | CLI-first, controller-per-resource (Kustomization, HelmRelease, GitRepository) |
| **Rancher Fleet** | SUSE/Rancher | Massive multi-cluster fleets (edge, retail) |
| **Codefresh GitOps** | Octopus (commercial) | ArgoCD under the hood + dashboards, drift, multi-team RBAC |
| **Jenkins X** | CD Foundation | If you must keep Jenkins |

| Инструмент | Происхождение | Сильная сторона |
|------|---------|-----------|
| **ArgoCD** | CNCF graduated (Intuit, 2018) | Приоритет UI, декларативный CRD `Application`, sync waves |
| **Flux CD v2** | CNCF graduated (Weaveworks, 2020) | Приоритет CLI, контроллер на каждый ресурс (Kustomization, HelmRelease, GitRepository) |
| **Rancher Fleet** | SUSE/Rancher | Огромные мультикластерные флоты (edge, retail) |
| **Codefresh GitOps** | Octopus (коммерческий) | ArgoCD под капотом + дашборды, дрейф, RBAC для многих команд |
| **Jenkins X** | CD Foundation | Если уж нужно сохранить Jenkins |

The course uses **ArgoCD** because (a) it's the most-adopted GitOps tool, (b) the UI makes drift visible to non-Kubernetes humans, and (c) the `Application` + `ApplicationSet` CRDs map cleanly onto multi-environment + multi-service teaching.

Курс использует **ArgoCD**, потому что (а) это самый распространённый GitOps-инструмент, (б) UI делает дрейф видимым для людей, далёких от Kubernetes, и (в) CRD `Application` + `ApplicationSet` напрямую ложатся на обучение по нескольким окружениям и сервисам.

> 🔥 **Honest take:** Flux is genuinely good — lighter, more Unix-y, no UI to keep alive. Pick it for a homelab. ArgoCD wins in enterprises because dashboards are how non-engineers tell that deploys worked.
>
> 🔥 **Честно:** Flux действительно хорош — легче, более «в духе Unix», нет UI, который надо поддерживать. Бери его для домашней лаборатории. ArgoCD побеждает в энтерпрайзе, потому что дашборды — это способ, которым не-инженеры понимают, что деплой прошёл.

---

## 📍 Slide 7 – 🏗️ ArgoCD Architecture

Слайд 7 — Архитектура ArgoCD

```mermaid
flowchart TD
  subgraph cluster["☸️ Kubernetes cluster (argocd namespace)"]
    API[🌐 argocd-server<br/>UI · gRPC · REST · webhook]
    Ctrl[🔄 application-controller<br/>StatefulSet · the reconciler]
    Repo[📦 repo-server<br/>clones Git · renders Helm/Kustomize]
    Redis[(⚡ redis<br/>manifest + cluster cache)]
    Dex[🔐 dex-server<br/>OIDC broker · optional]
    AS[♾️ applicationset-controller<br/>generates Apps from templates]
  end
  User[👨‍💻 User] -->|UI / CLI / API| API
  API <--> Redis
  Ctrl <--> Repo
  Ctrl <--> Redis
  Repo --> Git[📝 Git repo]
  Ctrl --> Kube[K8s API]
  AS --> Ctrl
  API -.->|SSO| Dex
```

Схема: пользователь обращается к argocd-server (UI, CLI, API), сервер работает с redis; application-controller общается с repo-server и redis, repo-server клонирует Git, контроллер ходит в K8s API, applicationset-controller передаёт приложения контроллеру, а server через SSO обращается к Dex.

* 🌐 **argocd-server** — UI + gRPC/REST API; what humans and CI talk to. Stateless, horizontally scalable.
  * 🌐 **argocd-server** — UI + gRPC/REST API; то, с чем общаются люди и CI. Stateless, масштабируется горизонтально.
* 🔄 **application-controller** — the heart. Reconciles every `Application` CR against the cluster. StatefulSet for sharding across many clusters.
  * 🔄 **application-controller** — сердце системы. Согласует каждый CR `Application` с кластером. StatefulSet для шардирования по множеству кластеров.
* 📦 **repo-server** — clones Git, runs `helm template` / `kustomize build`, returns rendered manifests to the controller. Stateless, cache-heavy.
  * 📦 **repo-server** — клонирует Git, выполняет `helm template` / `kustomize build`, возвращает отрендеренные манифесты контроллеру. Stateless, с активным кэшированием.
* ⚡ **redis** — caches manifests, cluster state, Git revisions. Lose it and reconciliation gets slow, not wrong.
  * ⚡ **redis** — кэширует манифесты, состояние кластера, ревизии Git. Если его потерять, согласование станет медленным, но не ошибочным.
* 🔐 **dex-server** — OIDC broker for GitHub/Google/Okta SSO. Optional; you can use local users for labs.
  * 🔐 **dex-server** — OIDC-брокер для SSO через GitHub/Google/Okta. Опционален; в лабах можно использовать локальных пользователей.
* ♾️ **applicationset-controller** — turns one template + a generator into many `Application` CRs.
  * ♾️ **applicationset-controller** — превращает один шаблон + генератор в множество CR `Application`.

---

## 📍 Slide 8 – 📝 The Application CRD

Слайд 8 — CRD Application

The atomic unit of ArgoCD is one `Application` — a pointer to "this folder in this repo at this revision goes into this namespace on this cluster".

Атомарная единица ArgoCD — один `Application`: указатель вида «эта папка в этом репозитории на этой ревизии идёт в этот namespace этого кластера».

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-python-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # 🗑️ cascade-delete K8s resources on app delete
spec:
  project: default
  source:
    repoURL: https://github.com/innodevops/student-repo.git
    targetRevision: main                         # 📌 branch, tag, or commit SHA
    path: k8s/charts/app-python
    helm:
      valueFiles:
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc        # in-cluster
    namespace: dev
  syncPolicy:
    automated:
      prune: true                                 # 🗑️ delete resources removed from Git
      selfHeal: true                              # 💚 revert manual cluster edits
    syncOptions:
      - CreateNamespace=true
```

Пояснение к коду: `Application` связывает `source` (репозиторий, ревизия, путь, values для Helm) с `destination` (кластер и namespace), а `syncPolicy.automated` включает автосинхронизацию с удалением лишнего (`prune`) и откатом ручных правок (`selfHeal`).

Key concepts:

Ключевые понятия:

* 📌 **`targetRevision: main`** vs **a SHA**: branch tracks moving HEAD (fine for dev), SHA pins exactly (use for prod releases).
  * 📌 **`targetRevision: main`** против **SHA:** ветка отслеживает движущийся HEAD (нормально для dev), SHA фиксирует точно (используй для продовых релизов).
* 🗑️ **`finalizers:`** without this, deleting the `Application` leaves K8s resources orphaned.
  * 🗑️ **`finalizers:`** без этого удаление `Application` оставит ресурсы K8s осиротевшими.
* 🔄 **`automated`** turns on auto-sync; *omitting it* makes the app **manual-sync** (click the button or `argocd app sync`).
  * 🔄 **`automated`** включает автосинхронизацию; если его *не указать*, приложение становится **manual-sync** (кнопка в UI или `argocd app sync`).

---

## 📍 Slide 9 – 🔄 Sync, Health, and Operation States

Слайд 9 — Состояния синхронизации, здоровья и операций

Every `Application` tracks two orthogonal states, plus a transient operation.

Каждый `Application` отслеживает два независимых состояния плюс текущую операцию.

| Sync state | Meaning |
|------------|---------|
| 🟢 **Synced** | Live cluster matches the rendered manifests |
| 🟡 **OutOfSync** | Drift between Git and cluster (good — you can see it!) |
| ❓ **Unknown** | Repo unreachable or controller hasn't checked yet |

| Состояние синхронизации | Значение |
|------------|---------|
| 🟢 **Synced** | Живой кластер совпадает с отрендеренными манифестами |
| 🟡 **OutOfSync** | Дрейф между Git и кластером (хорошо — это видно!) |
| ❓ **Unknown** | Репозиторий недоступен или контроллер ещё не проверял |

| Health state | Meaning |
|--------------|---------|
| 💚 **Healthy** | All resources report ready (Deployment available, Pod running, etc.) |
| 🔵 **Progressing** | Rollout in flight, give it a minute |
| 🔴 **Degraded** | Something failed (CrashLoopBackOff, schema error) |
| ⚪ **Suspended** | Hooks paused or app paused |
| 🚫 **Missing** | Resource not in cluster |

| Состояние здоровья | Значение |
|--------------|---------|
| 💚 **Healthy** | Все ресурсы готовы (Deployment доступен, Pod работает и т. д.) |
| 🔵 **Progressing** | Раскатка идёт, дай ей минуту |
| 🔴 **Degraded** | Что-то упало (CrashLoopBackOff, ошибка схемы) |
| ⚪ **Suspended** | Хуки или приложение на паузе |
| 🚫 **Missing** | Ресурса нет в кластере |

> 💡 An app can be **Synced + Degraded** (Git correctly says "deploy this broken Pod") or **OutOfSync + Healthy** (cluster runs fine but doesn't match Git). Both states are diagnostic.
>
> 💡 Приложение может быть **Synced + Degraded** (Git верно говорит «разверни этот сломанный Pod») или **OutOfSync + Healthy** (кластер работает нормально, но не совпадает с Git). Оба состояния диагностичны.

ArgoCD 3.4 added an **Operation Status** filter alongside Sync and Health, so the UI now shows "what's actively syncing right now" — useful during incident response.

В ArgoCD 3.4 появился фильтр **Operation Status** рядом с Sync и Health, так что UI теперь показывает, «что синхронизируется прямо сейчас» — полезно при разборе инцидентов.

---

## 📍 Slide 10 – 🎚️ Sync Policies: Manual, Auto, Self-Heal, Prune

Слайд 10 — Политики синхронизации: manual, auto, self-heal, prune

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
    allowEmpty: false       # 🛡️ refuse to apply if rendering produced 0 manifests
  syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
    - PruneLast=true
    - ApplyOutOfSyncOnly=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

Пояснение к коду: `automated` включает автосинхронизацию с `prune` и `selfHeal`, `allowEmpty: false` защищает от применения пустого результата рендера, `syncOptions` настраивают способ применения, а `retry` задаёт до 5 повторов с экспоненциальной задержкой.

* 🔄 **No `automated:` block → manual sync.** The UI shows a big "SYNC" button. Standard for prod.
  * 🔄 **Нет блока `automated:` → ручная синхронизация.** UI показывает большую кнопку «SYNC». Стандарт для прода.
* 🤖 **`automated:` → auto-sync** on every Git change *and* every periodic reconcile (default 3 min).
  * 🤖 **`automated:` → автосинхронизация** на каждое изменение в Git *и* на каждый периодический reconcile (по умолчанию раз в 3 минуты).
* 💚 **`selfHeal: true`** — revert manual cluster edits back to Git state. Disable if a controller owns the resource (e.g., HPA writing `replicas`).
  * 💚 **`selfHeal: true`** — откатывает ручные правки кластера обратно к состоянию Git. Отключай, если ресурсом владеет контроллер (например, HPA, который пишет `replicas`).
* 🗑️ **`prune: true`** — delete resources that were *removed* from Git. Without it, deleting a file in Git leaves an orphan running.
  * 🗑️ **`prune: true`** — удаляет ресурсы, которые *удалили* из Git. Без этого удаление файла в Git оставит работающую сироту.
* ⚙️ **`ServerSideApply=true`** uses K8s server-side apply (avoids the "last-applied-configuration" annotation problem and plays nice with HPA/VPA).
  * ⚙️ **`ServerSideApply=true`** использует server-side apply в K8s (избавляет от проблемы с аннотацией «last-applied-configuration» и хорошо уживается с HPA/VPA).

> ⚠️ **Sane default for prod:** manual sync, `prune: false`, `selfHeal: false`. Humans review every change. Dev environments turn everything on.
>
> ⚠️ **Разумный дефолт для прода:** ручная синхронизация, `prune: false`, `selfHeal: false`. Люди ревьюят каждое изменение. В dev-окружениях включают всё.

---

## 📍 Slide 11 – 🌊 Sync Waves and Hooks

Слайд 11 — Sync waves и хуки

Some resources must come up *before* others — CRDs before custom resources, namespaces before everything, DB migrations before the app.

Некоторые ресурсы должны подниматься *раньше* других — CRD до собственных ресурсов, namespace до всего остального, миграции БД до приложения.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"           # 🌊 negative = earlier, positive = later (default 0)
    argocd.argoproj.io/hook: PreSync             # 🪝 PreSync | Sync | PostSync | SyncFail | PostDelete
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

Пояснение к коду: аннотация `sync-wave` задаёт порядок применения (отрицательные значения — раньше, положительные — позже, по умолчанию 0), `hook` помечает ресурс как хук жизненного цикла, а `hook-delete-policy: HookSucceeded` удаляет его после успешного выполнения.

```mermaid
flowchart LR
  PreSync[🪝 PreSync hooks] --> W0[wave -1: CRDs]
  W0 --> W1[wave 0: namespaces, ConfigMaps]
  W1 --> W2[wave 1: Deployments]
  W2 --> W3[wave 2: Services, Ingress]
  W3 --> Post[🪝 PostSync hooks]
```

Схема: хуки PreSync выполняются до синхронизационной волны −1 (CRD), затем идут волна 0 (namespace, ConfigMap), волна 1 (Deployment), волна 2 (Service, Ingress), и в конце — хуки PostSync.

* 🪝 **Hooks** are arbitrary K8s resources (typically Jobs) that run at lifecycle points. Classic use: a `pre-sync` Job that runs `flyway migrate` before the new app pods start.
  * 🪝 **Хуки** — произвольные ресурсы K8s (обычно Jobs), которые выполняются в определённых точках жизненного цикла. Классика: Job `pre-sync`, который запускает `flyway migrate` до старта новых подов приложения.
* 🌊 **Waves** order Sync-phase resources. The controller applies everything at wave `N`, waits for health, then proceeds to `N+1`.
  * 🌊 **Волны** упорядочивают ресурсы фазы Sync. Контроллер применяет всё на волне `N`, ждёт готовности, затем переходит к `N+1`.
* 🧹 **`hook-delete-policy`** keeps the cluster tidy — `HookSucceeded` deletes the Job once it finishes green.
  * 🧹 **`hook-delete-policy`** держит кластер в чистоте — `HookSucceeded` удаляет Job после успешного завершения.

---

## 📍 Slide 12 – 🌳 The App-of-Apps Pattern

Слайд 12 — Паттерн App-of-Apps

Bootstrap problem: ArgoCD manages your apps via `Application` CRs in the `argocd` namespace. But who manages *those* `Application` CRs? Answer: another `Application`.

Проблема загрузки: ArgoCD управляет приложениями через CR `Application` в namespace `argocd`. Но кто управляет *этими* CR `Application`? Ответ: другой `Application`.

```
gitops-repo/
└── apps/
    ├── root.yaml              # 👈 the "App-of-Apps" — points at ./apps
    ├── app-python.yaml        # Application CR for the Python service
    ├── app-go-echo.yaml       # Application CR for the Go echo service
    └── app-go-health.yaml     # Application CR for the Go health service
```

Пояснение к коду: в папке `apps/` лежит корневой `root.yaml`, который указывает на неё же, и три YAML-файла `Application` для сервисов.

```yaml
# root.yaml — the seed
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/innodevops/student-repo.git
    targetRevision: main
    path: apps                                # 📂 directory full of Application YAMLs
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

Пояснение к коду: `root.yaml` — корневой `Application`, чей `path: apps` указывает на папку со всеми дочерними `Application`; `automated` включает синхронизацию с `prune` и `selfHeal`.

One `kubectl apply -f root.yaml` and ArgoCD discovers and syncs every child `Application`. The cluster is now self-managing — recovery = reinstall ArgoCD + reapply `root.yaml`.

Один `kubectl apply -f root.yaml` — и ArgoCD находит и синхронизирует каждый дочерний `Application`. Кластер теперь управляет собой сам: восстановление = переустановить ArgoCD + повторно применить `root.yaml`.

---

## 📍 Slide 13 – 🧬 The 3-Service ArgoCD Topology

Слайд 13 — Топология ArgoCD с тремя сервисами

```mermaid
flowchart TD
  Git[📝 GitHub repo<br/>k8s/charts + apps/]
  Git --> Root[🌱 root Application<br/>App-of-Apps]
  Root --> AS[♾️ ApplicationSet<br/>List generator: dev, prod]

  AS --> AP_D[🐍 app-python-dev]
  AS --> AG_D[🦫 app-go-echo-dev]
  AS --> AH_D[💚 app-go-health-dev]
  AS --> AP_P[🐍 app-python-prod]
  AS --> AG_P[🦫 app-go-echo-prod]
  AS --> AH_P[💚 app-go-health-prod]

  AP_D & AG_D & AH_D --> NSdev[☸️ namespace: dev]
  AP_P & AG_P & AH_P --> NSprod[☸️ namespace: prod]
```

Схема: 📝 GitHub-репозиторий с чартами и манифестами порождает 🌱 корневой App-of-Apps, тот создаёт ♾️ ApplicationSet с List-генератором на dev и prod, который разворачивает шесть приложений — по три на каждое окружение — в namespace `dev` и `prod`.

* 🐍 **app-python** — your Lab 2 → Lab 10 service (FastAPI).
  * 🐍 **app-python** — твой сервис из лаб 2 → 10 (FastAPI).
* 🦫 **app-go-echo** — Lab 9's plumbing companion (Go HTTP echo).
  * 🦫 **app-go-echo** — вспомогательный сервис из лабы 9 (HTTP-эхо на Go).
* 💚 **app-go-health** — **new in Lab 13** — a tiny Go health/static service shipped as plumbing. It exists *specifically* so that ApplicationSet and App-of-Apps stop being toy patterns.
  * 💚 **app-go-health** — **новый в лабе 13** — крошечный health/static-сервис на Go, поставляемый как «сантехника». Он существует *специально* для того, чтобы ApplicationSet и App-of-Apps перестали быть игрушечными паттернами.

> 💡 **Pedagogical point:** ApplicationSet with one service is just `kubectl apply` with extra steps. With **two** services in **two** environments it starts to earn its keep. With **three services × two environments = six Applications** generated from one 30-line template, the value is obvious. Hence Lab 13 introduces `app-go-health`.
>
> 💡 **Педагогический момент:** ApplicationSet с одним сервисом — это просто `kubectl apply` с лишними шагами. С **двумя** сервисами в **двух** окружениях он начинает оправдывать себя. А с **тремя сервисами × двумя окружениями = шестью Applications**, сгенерированными из одного шаблона на 30 строк, ценность очевидна. Поэтому лаба 13 вводит `app-go-health`.

---

## 📍 Slide 14 – ♾️ ApplicationSet: One Template, Many Apps

Слайд 14 — ApplicationSet: один шаблон, много приложений

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-apps
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:                       # 1️⃣ which apps
              elements:
                - { svc: app-python,     path: k8s/charts/app-python }
                - { svc: app-go-echo,    path: k8s/charts/app-go-echo }
                - { svc: app-go-health,  path: k8s/charts/app-go-health }
          - list:                       # 2️⃣ which environments
              elements:
                - { env: dev,  autoSync: "true" }
                - { env: prod, autoSync: "false" }
  template:
    metadata:
      name: '{{svc}}-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/innodevops/student-repo.git
        targetRevision: main
        path: '{{path}}'
        helm:
          valueFiles:
            - 'values-{{env}}.yaml'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{env}}'
      syncPolicy:
        syncOptions: [CreateNamespace=true]
```

Пояснение к коду: генератор `matrix` перемножает два `list`-генератора — список сервисов и список окружений; шаблон `template` подставляет `{{svc}}`, `{{env}}` и `{{path}}`, создавая по `Application` на каждую пару.

The **matrix** generator cross-joins two **list** generators → 3 services × 2 envs = **6 `Application` CRs** materialized automatically. Add a fourth service? One line in the first list.

Генератор **matrix** перемножает два генератора **list** → 3 сервиса × 2 окружения = **6 CR `Application`**, созданных автоматически. Добавить четвёртый сервис? Одна строка в первом списке.

---

## 📍 Slide 15 – 🧰 The Other ApplicationSet Generators

Слайд 15 — Остальные генераторы ApplicationSet

| Generator | What it iterates over | Use when |
|-----------|----------------------|----------|
| **List** | Inline literals | Small, stable set (dev/staging/prod) |
| **Cluster** | All clusters registered with ArgoCD | Multi-cluster fleet rollouts |
| **Git (Directory)** | Folders under a path in a repo | One folder per app — auto-discover new apps |
| **Git (File)** | YAML/JSON files at a path | One config file per tenant |
| **Matrix** | Cross-join of two generators | Apps × environments, services × clusters |
| **Merge** | Inner-join + override | Layer cluster-specific overrides onto a base list |
| **SCM Provider** | Repos in a GitHub/GitLab org | Org-wide GitOps for many service repos |
| **Pull Request** | Open PRs in a repo | Ephemeral preview environments per PR |
| **Cluster Decision** | A CRD that picks clusters | External logic decides where apps land |
| **Plugin** | An HTTP service you write | Anything the above can't express |

| Генератор | По чему итерируется | Когда использовать |
|-----------|----------------------|----------|
| **List** | Встроенные литералы | Небольшой стабильный набор (dev/staging/prod) |
| **Cluster** | Все кластеры, зарегистрированные в ArgoCD | Раскатки на мультикластерный флот |
| **Git (Directory)** | Папки по пути в репозитории | Одна папка на приложение — автообнаружение новых приложений |
| **Git (File)** | YAML/JSON-файлы по пути | Один конфиг-файл на тенанта |
| **Matrix** | Перемножение двух генераторов | Приложения × окружения, сервисы × кластеры |
| **Merge** | Inner join + переопределения | Наложение кластерных переопределений на базовый список |
| **SCM Provider** | Репозитории в организации GitHub/GitLab | GitOps для всей организации с множеством сервисных репозиториев |
| **Pull Request** | Открытые PR в репозитории | Временные preview-окружения на каждый PR |
| **Cluster Decision** | CRD, который выбирает кластеры | Внешняя логика решает, куда попадут приложения |
| **Plugin** | HTTP-сервис, который ты пишешь | Всё, что не выразить вышеперечисленным |

> 💡 **Lab 13 uses List + Matrix.** Bonus: try the Git Directory generator pointed at `k8s/charts/` — adding a new chart folder automatically creates an Application.
>
> 💡 **В лабе 13 используются List + Matrix.** Бонус: попробуй генератор Git Directory, направленный на `k8s/charts/` — добавление новой папки с чартом автоматически создаст Application.

---

## 📍 Slide 16 – 🔐 RBAC and AppProjects

Слайд 16 — RBAC и AppProjects

Out of the box, every `Application` lands in the `default` project, which can deploy anywhere. In production, you carve up access with **AppProject** CRs.

По умолчанию каждый `Application` попадает в проект `default`, который может деплоить куда угодно. В продакшене доступ разграничивают через CR **AppProject**.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-payments
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/innodevops/payments-*    # 🔒 only these repos
  destinations:
    - namespace: payments-*                       # 🔒 only these namespaces
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:                       # 🔒 only these cluster-scoped kinds
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  roles:
    - name: deployer
      policies:
        - p, proj:team-payments:deployer, applications, sync, team-payments/*, allow
      groups: [innodevops:payments]
```

Пояснение к коду: `sourceRepos` и `destinations` ограничивают репозитории и namespace, `clusterResourceWhitelist` — кластерные ресурсы, `namespaceResourceBlacklist` запрещает отдельные kinds, а `roles` с политикой `p, proj:...` привязывают права к OIDC-группе.

The user-facing **argocd-rbac-cm** ConfigMap maps OIDC groups → roles → permissions. Pattern: one project per team, no `default` use in prod, OIDC groups for human role bindings.

Пользовательский ConfigMap **argocd-rbac-cm** сопоставляет OIDC-группы → роли → разрешения. Паттерн: один проект на команду, никакого `default` в проде, OIDC-группы для привязки ролей к людям.

---

## 📍 Slide 17 – 🔒 Secrets in a GitOps World

Слайд 17 — Секреты в мире GitOps

GitOps demands "everything in Git" — but plaintext secrets in Git are an immediate breach. Three production-grade answers:

GitOps требует «всё в Git» — но секреты в открытом виде в Git — это немедленная утечка. Три решения продакшен-уровня:

| Pattern | How it works | Tradeoff |
|---------|--------------|----------|
| **Sealed Secrets** (Bitnami) | Controller in cluster has a keypair; you `kubeseal` a `Secret` into a `SealedSecret` (cipher in YAML); controller decrypts on apply | Per-cluster key — re-seal on rotation |
| **External Secrets Operator + OpenBao** | Git holds only an `ExternalSecret` *reference*; ESO pulls the actual value from OpenBao/Vault/SSM at sync time | Operational dependency on the secret store |
| **SOPS + KMS / age** | Files encrypted in Git with `sops`; ArgoCD plugin (`argocd-vault-plugin`, `helm-secrets`) decrypts at render time | Plugin install + key management |

| Паттерн | Как работает | Компромисс |
|---------|--------------|----------|
| **Sealed Secrets** (Bitnami) | Контроллер в кластере хранит пару ключей; ты `kubeseal`-ишь `Secret` в `SealedSecret` (шифр в YAML); контроллер расшифровывает при применении | Ключ на каждый кластер — перезапечатывать при ротации |
| **External Secrets Operator + OpenBao** | В Git лежит только *ссылка* `ExternalSecret`; ESO забирает реальное значение из OpenBao/Vault/SSM во время синхронизации | Операционная зависимость от хранилища секретов |
| **SOPS + KMS / age** | Файлы шифруются в Git через `sops`; плагин ArgoCD (`argocd-vault-plugin`, `helm-secrets`) расшифровывает при рендере | Установка плагинов + управление ключами |

Your **Lab 11** already wired OpenBao + ESO — that's the pattern Lab 13 keeps using. The `Secret` resources never appear in Git; only `ExternalSecret` references do.

В твоей **лабе 11** уже подключены OpenBao + ESO — этот паттерн лаба 13 продолжает использовать. Ресурсы `Secret` никогда не появляются в Git; только ссылки `ExternalSecret`.

> 🔥 **Never** put plaintext secrets in the GitOps repo, even private. Git history is forever.
>
> 🔥 **Никогда** не клади секреты в открытом виде в GitOps-репозиторий, даже приватный. История Git вечна.

---

## 📍 Slide 18 – 🌐 Multi-Cluster GitOps

Слайд 18 — Мультикластерный GitOps

ArgoCD running in **one** "control plane" cluster can manage **many** workload clusters.

ArgoCD в **одном** кластере «control plane» может управлять **множеством** рабочих кластеров.

```mermaid
flowchart LR
  Git[📝 Git] --> Argo[🤖 ArgoCD<br/>control-plane cluster]
  Argo -->|kubeconfig in Secret| C1[☸️ dev cluster]
  Argo -->|kubeconfig in Secret| C2[☸️ staging cluster]
  Argo -->|kubeconfig in Secret| C3[☸️ prod cluster]
  Argo -->|kubeconfig in Secret| C4[☸️ edge cluster]
```

Схема: ArgoCD из control-plane-кластера через kubeconfig в Secret управляет dev-, staging-, prod- и edge-кластерами, а единственный источник — 📝 Git.

* 🔑 Each target cluster is registered with `argocd cluster add <context>`, which creates a `Secret` with its kubeconfig in the `argocd` namespace.
  * 🔑 Каждый целевой кластер регистрируется командой `argocd cluster add <context>`, которая создаёт `Secret` с его kubeconfig в namespace `argocd`.
* ♾️ The **Cluster generator** in an ApplicationSet pairs this with one template → "deploy this app to every cluster labelled `tier=edge`".
  * ♾️ **Генератор Cluster** в ApplicationSet сочетает это с одним шаблоном → «разверни это приложение во всех кластерах с меткой `tier=edge`».
* ⏸️ **ArgoCD 3.4 added Pause Reconciliation per cluster** — flip a toggle to freeze a misbehaving cluster mid-incident without touching others.
  * ⏸️ **В ArgoCD 3.4 добавили Pause Reconciliation на уровне кластера** — переключи тумблер, чтобы заморозить проблемный кластер прямо во время инцидента, не трогая остальные.

> 💡 **Hub-and-spoke vs federated:** the hub model is simpler; for >50 clusters or strict network isolation, consider **Argo CD Argocd in each cluster** + a higher-level fleet controller (Fleet, Cluster API).
>
> 💡 **Hub-and-spoke против федеративного:** модель хаба проще; для >50 кластеров или строгой сетевой изоляции рассмотри **Argo CD в каждом кластере** + fleet-контроллер верхнего уровня (Fleet, Cluster API).

---

## 📍 Slide 19 – 🩺 Observing ArgoCD Itself

Слайд 19 — Наблюдение за самим ArgoCD

ArgoCD ships Prometheus metrics out of the box (your Lab 8 stack scrapes them).

ArgoCD из коробки отдаёт метрики Prometheus (твой стек из лабы 8 их скрейпит).

| Metric | Watch for |
|--------|-----------|
| `argocd_app_info{sync_status="OutOfSync"}` | Persistent drift |
| `argocd_app_info{health_status="Degraded"}` | Broken apps |
| `argocd_app_sync_total{phase="Failed"}` | Failing syncs |
| `argocd_app_reconcile_bucket` | Reconcile latency histogram |
| `argocd_cluster_api_resource_objects` | Object count growth (scaling signal) |
| `argocd_redis_request_total` | Cache load |

| Метрика | За чем следить |
|--------|-----------|
| `argocd_app_info{sync_status="OutOfSync"}` | Устойчивый дрейф |
| `argocd_app_info{health_status="Degraded"}` | Сломанные приложения |
| `argocd_app_sync_total{phase="Failed"}` | Неудачные синхронизации |
| `argocd_app_reconcile_bucket` | Гистограмма задержки согласования |
| `argocd_cluster_api_resource_objects` | Рост числа объектов (сигнал масштабирования) |
| `argocd_redis_request_total` | Нагрузка на кэш |

Wire these into your Lab 8 Grafana stack and alert on **Degraded > 5 minutes** and **OutOfSync > 30 minutes** (auto-sync clusters should never sit OutOfSync that long; if they do, something is wedged).

Подключи их к стеку Grafana из лабы 8 и настрой алерты на **Degraded > 5 минут** и **OutOfSync > 30 минут** (кластеры с автосинхронизацией не должны оставаться OutOfSync так долго; если остаются — что-то заклинило).

---

## 📍 Slide 20 – 🚨 Disaster Recovery: Git Is Your Backup

Слайд 20 — Аварийное восстановление: Git — твой бэкап

The clean GitOps recovery story:

Чистая история восстановления в GitOps:

```mermaid
flowchart LR
  Dead[💀 Cluster gone] --> New[🆕 Fresh cluster]
  New -->|helm install argo-cd| Argo[🤖 ArgoCD]
  Argo -->|kubectl apply -f root.yaml| Root[🌱 App-of-Apps]
  Root --> All[♾️ All apps reconciled]
  All --> Back[✅ Back online]
```

Схема: 💀 кластер пропал → 🆕 новый кластер → установка ArgoCD через Helm → применение `root.yaml` → 🌱 App-of-Apps → все приложения согласованы → ✅ снова в строю.

1. 🆕 Provision a new cluster (Terraform / kubeadm / kOps).
   * 🆕 Подготовить новый кластер (Terraform / kubeadm / kOps).
2. 📦 `helm install argo-cd argo/argo-cd -n argocd --create-namespace`.
   * 📦 `helm install argo-cd argo/argo-cd -n argocd --create-namespace`.
3. 🌱 `kubectl apply -f apps/root.yaml`.
   * 🌱 `kubectl apply -f apps/root.yaml`.
4. ☕ Wait. Everything else is in Git.
   * ☕ Подождать. Всё остальное уже в Git.

> ⚠️ **What's *not* recovered:** stateful data (PVCs, databases). GitOps restores **configuration**, not **state**. Pair with PV snapshots / Velero / database backups. This is the same lesson Labs 11 (secrets) and 12 (PVCs) hammered.
>
> ⚠️ **Что *не* восстанавливается:** данные с состоянием (PVC, базы данных). GitOps восстанавливает **конфигурацию**, а не **данные**. Сочетай со снапшотами PV / Velero / бэкапами БД. Это тот же урок, который вбивали лабы 11 (секреты) и 12 (PVC).

---

## 📍 Slide 21 – 🧰 Anti-Patterns and Common Bugs

Слайд 21 — Антипаттерны и типичные баги

1. ❌ **`kubectl apply` to a GitOps-managed namespace** — self-heal reverts it in 3 minutes and you wasted everyone's afternoon debugging. Make the change in Git.
   * ❌ **`kubectl apply` в namespace, управляемый GitOps** — self-heal откатит это через 3 минуты, и ты потратишь всем день на отладку. Делай изменение в Git.
2. ❌ **`prune: true` on first install** — if your `path:` is wrong, ArgoCD happily deletes everything it doesn't see. Start with `prune: false`, verify, then turn it on.
   * ❌ **`prune: true` при первой установке** — если твой `path:` указан неверно, ArgoCD с радостью удалит всё, чего не видит. Начни с `prune: false`, проверь, потом включай.
3. ❌ **Pointing `targetRevision` at `HEAD`** for prod — silent rollouts on every commit to `main`. Use a tag or `release/*` branch.
   * ❌ **`targetRevision` на `HEAD`** для прода — тихие раскатки на каждый коммит в `main`. Используй тег или ветку `release/*`.
4. ❌ **Storing plaintext `Secret` YAML in Git** — even in a private repo. Use Sealed Secrets / ESO / SOPS.
   * ❌ **Хранение `Secret` YAML в открытом виде в Git** — даже в приватном репозитории. Используй Sealed Secrets / ESO / SOPS.
5. ❌ **One giant `Application` for "everything"** — you lose per-app sync, per-app status, per-app RBAC. One Application per (service × env).
   * ❌ **Один гигантский `Application` на «всё»** — теряешь синхронизацию, статус и RBAC по каждому приложению. Один Application на пару (сервис × окружение).
6. ❌ **Mixing repo concerns** — keep **app code** and **GitOps manifests** in *different* repos (or at least different paths). CI builds the image; GitOps repo references the new tag.
   * ❌ **Смешивание ответственности репозиториев** — держи **код приложения** и **GitOps-манифесты** в *разных* репозиториях (или как минимум по разным путям). CI собирает образ; GitOps-репозиторий ссылается на новый тег.
7. ❌ **Forgetting the finalizer** — orphan resources on `kubectl delete app` are how rogue Pods survive in production for a year.
   * ❌ **Забыть finalizer** — осиротевшие ресурсы после `kubectl delete app` — так в проде и живут бродячие Pod'ы по году.
8. ❌ **selfHeal on resources with HPA** — the HPA writes `spec.replicas`, ArgoCD reverts it, fight ensues. Use `ignoreDifferences:` on `/spec/replicas`.
   * ❌ **selfHeal на ресурсах с HPA** — HPA пишет `spec.replicas`, ArgoCD откатывает, начинается борьба. Используй `ignoreDifferences:` на `/spec/replicas`.

---

## 📍 Slide 22 – 🌍 GitOps in the Wild

Слайд 22 — GitOps в реальном мире

* 🏢 **Intuit** open-sourced ArgoCD in 2018 to manage ~2,000 microservices; they run hundreds of deploys/day off Git.
  * 🏢 **Intuit** открыл исходники ArgoCD в 2018 году, чтобы управлять ~2000 микросервисами; они делают сотни деплоев в день из Git.
* 🛒 **BlackRock, Adobe, Tesla, Red Hat OpenShift GitOps** all ship ArgoCD in production at large scale.
  * 🛒 **BlackRock, Adobe, Tesla, Red Hat OpenShift GitOps** — все используют ArgoCD в проде в больших масштабах.
* 📊 **CNCF Annual Survey 2024:** GitOps adoption is the #1-growing K8s practice; **ArgoCD** is the dominant tool in survey responses.
  * 📊 **Годовой опрос CNCF 2024:** внедрение GitOps — практика K8s с самым быстрым ростом; **ArgoCD** — доминирующий инструмент в ответах.
* 🏷️ **Argo project graduated CNCF** in December 2022 — same maturity tier as Kubernetes, Prometheus, Envoy.
  * 🏷️ **Проект Argo получил статус graduated CNCF** в декабре 2022 — тот же уровень зрелости, что у Kubernetes, Prometheus, Envoy.
* 🚀 **ArgoCD 3.4** (early May 2026) doubled down on Day-2 ops: pause-per-cluster, richer UI filters, K8s version stored as Major.Minor.Patch, OpenTelemetry tracing through the OIDC flow.
  * 🚀 **ArgoCD 3.4** (начало мая 2026) усилил Day-2 эксплуатацию: пауза на уровне кластера, более богатые фильтры в UI, версия K8s в формате Major.Minor.Patch, трассировка OpenTelemetry в OIDC-потоке.

> 📊 **One number:** Codefresh's 2024 "State of GitOps" survey put **ArgoCD at ~70% of GitOps deployments**, Flux at ~25%, the rest in the long tail.
>
> 📊 **Одна цифра:** опрос Codefresh «State of GitOps» 2024 года оценил **ArgoCD примерно в 70% GitOps-деплоев**, Flux — около 25%, остальное — длинный хвост.

---

## 📍 Slide 23 – 🎯 Key Takeaways

Слайд 23 — Главное из лекции

1. 📜 **GitOps = the four OpenGitOps principles** — declarative, versioned, pulled, continuously reconciled. Everything else is implementation detail.
   * 📜 **GitOps = четыре принципа OpenGitOps** — декларативность, версионирование, автоматическое вытягивание, непрерывное согласование. Всё остальное — детали реализации.
2. 🚀 **Pull beats push** on credential surface, drift detection, and disaster recovery.
   * 🚀 **Pull лучше push** по поверхности учётных данных, обнаружению дрейфа и аварийному восстановлению.
3. 🏗️ **ArgoCD = 5 components** — argocd-server, application-controller, repo-server, redis, dex (+ applicationset-controller). Memorize their jobs.
   * 🏗️ **ArgoCD = 5 компонентов** — argocd-server, application-controller, repo-server, redis, dex (+ applicationset-controller). Запомни их задачи.
4. 📝 **Application CRD** is the atomic unit; **App-of-Apps** bootstraps the whole cluster from one seed.
   * 📝 **CRD Application** — атомарная единица; **App-of-Apps** загружает весь кластер из одного seed-манифеста.
5. ♾️ **ApplicationSet** with Matrix(List × List) generates N services × M environments from one template — the lab pattern.
   * ♾️ **ApplicationSet** с Matrix(List × List) генерирует N сервисов × M окружений из одного шаблона — паттерн лабы.
6. 🎚️ **Sync policy = manual for prod, automated+selfHeal+prune for dev** is a sane default.
   * 🎚️ **Политика синхронизации = ручная для прода, automated+selfHeal+prune для dev** — разумный дефолт.
7. 🌊 **Sync waves + hooks** order DB migrations before app pods and cleanup jobs after.
   * 🌊 **Sync waves + хуки** упорядочивают миграции БД до подов приложения и задачи очистки после.
8. 🔒 **Secrets stay out of Git** — Sealed Secrets / ESO + OpenBao / SOPS. Forever.
   * 🔒 **Секреты вне Git** — Sealed Secrets / ESO + OpenBao / SOPS. Навсегда.

> 💬 *"Operations by Pull Request."* — Kelsey Hightower
>
> 💬 *«Operations by Pull Request»* — Келси Хайтауэр

---

## 📍 Slide 24 – 🚀 What Comes Next

Слайд 24 — Что дальше

**📚 Next lecture: *Progressive Delivery with Argo Rollouts*** — because once GitOps deploys instantly on merge, the next question is *"how do we deploy **carefully**?"* Canary, blue-green, automated analysis.

**📚 Следующая лекция: *Progressive Delivery с Argo Rollouts*** — потому что когда GitOps деплоит моментально по мержу, следующий вопрос: *«как деплоить **аккуратно**?»* Canary, blue-green, автоматический анализ.

* 🐤 **Canary deployments** — 10% → 50% → 100%, gated by Prometheus queries
  * 🐤 **Canary-деплои** — 10% → 50% → 100%, с проверками по запросам Prometheus
* 🔵 **Blue-green** — two full stacks, instant switch, instant rollback
  * 🔵 **Blue-green** — два полных стека, мгновенное переключение, мгновенный откат
* 📊 **AnalysisTemplate** — metric-driven promotion / rollback
  * 📊 **AnalysisTemplate** — продвижение / откат по метрикам
* 🔄 **Argo Rollouts** vs Flagger — the two contenders
  * 🔄 **Argo Rollouts** против Flagger — два конкурента

**🔬 Lab 13 deliverables:**

**🔬 Что сдаёшь в лабе 13:**

* Install ArgoCD 3.4 via Helm in your cluster
  * Установить ArgoCD 3.4 через Helm в свой кластер
* Add a **third service** — `app-go-health` (plumbing files shipped in `app_go_health/`)
  * Добавить **третий сервис** — `app-go-health` (вспомогательные файлы в `app_go_health/`)
* Declare an `Application` for each of the **three** services, in **two** environments (dev + prod) — six Applications total
  * Объявить `Application` для каждого из **трёх** сервисов в **двух** окружениях (dev + prod) — шесть Applications всего
* Replace the six with one **ApplicationSet** using a Matrix(List, List) generator
  * Заменить шесть на один **ApplicationSet** с генератором Matrix(List, List)
* Wrap the lot in an **App-of-Apps** root Application
  * Обернуть всё в корневой **App-of-Apps** Application
* Auto-sync dev with selfHeal + prune; keep prod **manual**
  * Включить автосинхронизацию для dev с selfHeal + prune; прод оставить **ручным**
* Bonus 2.5 pts: switch to the Git Directory generator that auto-discovers new charts under `k8s/charts/*`
  * Бонус 2.5 балла: перейти на генератор Git Directory, который автоматически находит новые чарты в `k8s/charts/*`

```mermaid
flowchart LR
  Lab10[📦 Lab 10: Helm chart] --> Lab13[🤖 Lab 13: ArgoCD<br/>+ 3rd service]
  Lab13 --> Lab14[🚀 Lab 14: Argo Rollouts<br/>progressive delivery]
```

Схема: 📦 Helm-чарт из лабы 10 → 🤖 ArgoCD из лабы 13 с третьим сервисом → 🚀 Argo Rollouts из лабы 14 с progressive delivery.

> 🌊 From "I deployed it" to "Git deployed it" — one merge at a time.
>
> 🌊 От «я это задеплоил» к «Git это задеплоил» — по одному мержу за раз.

---

## 📚 Resources

Материалы

* 📕 *GitOps and Kubernetes* — Yuen, Matyushentsev, Ekenstam, Suen (Manning, 2021) — canonical book on the pattern
  * 📕 *GitOps and Kubernetes* — Юэн, Матюшенцев, Экенстам, Суэн (Manning, 2021) — каноническая книга о паттерне
* 📕 *The Path to GitOps* — Christian Hernandez (Red Hat e-book, 2024) — short, practitioner-focused
  * 📕 *The Path to GitOps* — Кристиан Эрнандес (e-book Red Hat, 2024) — коротко и по делу для практиков
* 🌐 [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/stable/) — official ArgoCD docs
  * 🌐 [argo-cd.readthedocs.io](https://argo-cd.readthedocs.io/en/stable/) — официальная документация ArgoCD
* 🌐 [opengitops.dev](https://opengitops.dev/) — the four principles, ratified
  * 🌐 [opengitops.dev](https://opengitops.dev/) — четыре принципа в утверждённом виде
* 🌐 [github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd) — source + releases
  * 🌐 [github.com/argoproj/argo-cd](https://github.com/argoproj/argo-cd) — исходники + релизы
* 🌐 [fluxcd.io](https://fluxcd.io/) — the other CNCF GitOps tool worth knowing
  * 🌐 [fluxcd.io](https://fluxcd.io/) — второй GitOps-инструмент CNCF, который стоит знать
* 🌐 [Codefresh "State of GitOps" survey](https://codefresh.io/) — annual adoption numbers
  * 🌐 [Опрос Codefresh «State of GitOps»](https://codefresh.io/) — ежегодные цифры внедрения
* 🎙️ KubeCon 2023 keynote — *"GitOps at Intuit"* (Hong Wang) — origin story of ArgoCD at scale
  * 🎙️ Кейноут KubeCon 2023 — *«GitOps at Intuit»* (Хонг Ван) — история ArgoCD в масштабе

**🎓 Quiz:** Post-lecture quiz feeds the weeks 13-16 leaderboard window.

**🎓 Квиз:** квиз после лекции идёт в зачёт лидерборда недель 13–16.
