# Lab 15 — StatefulSets & Persistent Storage

Лаба 15 — StatefulSets и постоянное хранилище

![difficulty](https://img.shields.io/badge/difficulty-advanced-red)
![topic](https://img.shields.io/badge/topic-StatefulSets-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-StatefulSet%20%7C%20PVC%20%7C%20K8s%201.36-informational)

> **Goal:** Run the workloads that *remember*. Give each pod a stable name and its own disk, prove the data on `web-1` follows `web-1` (not a random pod) across a `kubectl delete`.
> **Deliverable:** A PR from `lab15` converting your Lab 12 chart from a Deployment to a StatefulSet + headless Service, with `k8s/docs/LAB15.md` containing the per-pod-PVC persistence proof (write to `web-1`, kill `web-1`, read same value back; write to `web-0`, prove the values don't cross).
>
> **Цель:** запустить нагрузки, которые *помнят*. Дать каждому поду стабильное имя и собственный диск и доказать, что данные на `web-1` следуют за `web-1` (а не за случайным подом) после `kubectl delete`.
> **Что сдать:** PR из ветки `lab15`, где твой чарт из лабы 12 переведён с Deployment на StatefulSet + headless Service, а в `k8s/docs/LAB15.md` лежит доказательство сохранности PVC для каждого пода (записать в `web-1`, убить `web-1`, прочитать то же значение; записать в `web-0` и доказать, что значения не пересекаются).

---

## Overview

Обзор

Through Lab 14 every pod was disposable cattle — kill it, get a fresh one with a random name and an empty disk, nobody noticed. That breaks the moment a workload has to remember something *per pod* — a database replica, a queue partition, a search shard. A `Deployment` gives interchangeable pods with shared (or no) storage; a **StatefulSet** gives each pod a stable ordinal name, its own `PersistentVolumeClaim`, and an ordered lifecycle.

До лабы 14 включительно каждый под был одноразовым скотом: убей его — получишь новый со случайным именем и пустым диском, и никто этого не заметит. Это ломается в тот момент, когда нагрузке нужно помнить что-то *для каждого пода*: реплика базы данных, партиция очереди, шард поиска. `Deployment` даёт взаимозаменяемые поды с общим (или вовсе без) хранилищем; **StatefulSet** даёт каждому поду стабильное порядковое имя, собственный `PersistentVolumeClaim` и упорядоченный жизненный цикл.

In this lab you will practice:

В этой лабе ты отработаешь:

- **Writing a StatefulSet manifest from the API up** — `apiVersion`, `serviceName`, `replicas`, the label triangle, the pod template, and the `volumeClaimTemplates` list
  * **Написание манифеста StatefulSet начиная с API** — `apiVersion`, `serviceName`, `replicas`, треугольник лейблов, шаблон пода и список `volumeClaimTemplates`
- **Writing a headless Service from scratch** — and understanding *why* `clusterIP: None` is the thing that makes per-pod DNS exist
  * **Написание headless Service с нуля** — и понимание того, *почему* именно `clusterIP: None` создаёт DNS для отдельных подов
- The headline proof: **write a marker into `web-1`'s volume → `kubectl delete pod web-1` → read the marker back from the new `web-1` → it MATCHES** because `data-web-1` is bound to ordinal 1, not to the pod instance
  * Главное доказательство: **записать маркер в том `web-1` → `kubectl delete pod web-1` → прочитать маркер из нового `web-1` → он СОВПАДАЕТ**, потому что `data-web-1` привязан к порядковому номеру 1, а не к экземпляру пода
- Cross-checking that `web-0`'s data is *its own*, not `web-1`'s — that's the second half of the proof
  * Перекрёстная проверка, что данные `web-0` — *его собственные*, а не `web-1`: это вторая половина доказательства

> ⚠️ **Scope:** k3d's default `local-path` StorageClass, one StatefulSet, three replicas, `ReadWriteOnce` access. No multi-AZ, no operators, no failover. The Bonus introduces update strategies — pick one of three paths.
>
> ⚠️ **Границы:** StorageClass `local-path` по умолчанию в k3d, один StatefulSet, три реплики, режим доступа `ReadWriteOnce`. Никаких мульти-AZ, операторов и фейловера. В бонусном задании разбираются стратегии обновления — выбери один из трёх путей.

> 🪨 **Pedagogical core.** Lab 12 proved a PVC survives a pod kill. That used **one shared PVC** for one Deployment replica. Lab 15 proves something deeper: with `volumeClaimTemplates`, each ordinal `N` is bound to its own PVC `data-<sts>-N`, and that binding is sticky across reschedule, restart, even node failure. **If only one capture survives, capture the write-delete-read on `web-1` showing the marker came back.**
>
> 🪨 **Педагогическая суть.** Лаба 12 доказала, что PVC переживает убийство пода. Там использовался **один общий PVC** для одной реплики Deployment. Лаба 15 доказывает более глубокую вещь: с `volumeClaimTemplates` каждый порядковый номер `N` привязан к собственному PVC `data-<sts>-N`, и эта привязка сохраняется при пересоздании, перезапуске и даже отказе узла. **Если удастся сохранить только один скриншот, сохрани запись-удаление-чтение на `web-1`, где маркер вернулся.**

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно быть у тебя после предыдущих лаб:**

- Lab 9 — your `web` Deployment + Service on k3d (1.36) with 3 nodes
  * Лаба 9 — твои `web` Deployment + Service в k3d (1.36) с тремя узлами
- Lab 10 — your `lab10-app` Helm chart
  * Лаба 10 — твой Helm-чарт `lab10-app`
- Lab 11 — Secrets wired in
  * Лаба 11 — подключённые Secrets
- Lab 12 — the `/visits` counter writing to `$DATA_DIR/visits`, with a PVC mounted at that path, and the write-delete-read persistence proof
  * Лаба 12 — счётчик `/visits`, пишущий в `$DATA_DIR/visits`, с PVC, смонтированным по этому пути, и доказательством сохранности по схеме запись-удаление-чтение

**This lab adds (in your existing chart):**

**Эта лаба добавляет (в твой существующий чарт):**

- `templates/statefulset.yaml` — **you write** (replaces the Deployment for this lab; keep `deployment.yaml` in-tree but gate one off via values)
  * `templates/statefulset.yaml` — **пишешь ты** (заменяет Deployment в этой лабе; `deployment.yaml` оставь в репозитории, но отключи через values)
- `templates/service-headless.yaml` — **you write** (alongside the regular Service from Lab 9; both coexist)
  * `templates/service-headless.yaml` — **пишешь ты** (рядом с обычным Service из лабы 9; оба сосуществуют)
- A small `values.yaml` change to flip from `Deployment` to `StatefulSet` and set `replicaCount: 3`
  * Небольшое изменение `values.yaml`, чтобы переключиться с `Deployment` на `StatefulSet` и задать `replicaCount: 3`
- `k8s/docs/LAB15.md` — your submission report
  * `k8s/docs/LAB15.md` — твой отчёт о сдаче

You **do not** remove the existing regular Service (the ClusterIP one from Lab 9). Keep both: the regular Service load-balances across pods for any client that doesn't care which one answers; the headless Service is for per-pod addressing.

Ты **не** удаляешь существующий обычный Service (тот, что с ClusterIP, из лабы 9). Оставь оба: обычный Service балансирует нагрузку между подами для любого клиента, которому всё равно, какой из них ответит; headless Service нужен для адресации отдельных подов.

> 📚 Pairs with **Lecture 15 — StatefulSets & Persistent Storage**. Re-read slides 5 (the three guarantees), 6–7 (headless Services + per-pod DNS), 8 (`volumeClaimTemplates`), and 19 (the orphaned-PVC trap) before you start.
>
> 📚 В паре с **лекцией 15 — StatefulSets и постоянное хранилище**. Перед началом перечитай слайды 5 (три гарантии), 6–7 (headless Services и DNS для отдельных подов), 8 (`volumeClaimTemplates`) и 19 (ловушка с осиротевшими PVC).

---

## Setup

Подготовка

You need the k3d cluster from Lab 9 running on Kubernetes **1.36**, your `lab10-app` chart, and `kubectl`.

Тебе понадобится кластер k3d из лабы 9 на Kubernetes **1.36**, твой чарт `lab10-app` и `kubectl`.

```bash
kubectl get nodes                       # k3d-devops-server-0 + 2 agents on v1.36.x
kubectl get storageclass                # local-path (default) — what your PVCs will use
helm list -n lab15 2>/dev/null
```

Пояснение: первая команда показывает узлы кластера, вторая — доступные StorageClass, третья — релизы Helm в пространстве имён `lab15` (ошибки подавляются).

> 💡 The cluster note from Lab 12 still applies: k3d ships `local-path` (`rancher.io/local-path`) as the default `StorageClass`, marked `(default)`. It's RWO-only, dynamically-provisioned via hostPath under the hood, perfect for a 3-replica counter — and identical in *interface* to EBS / GCE PD / Ceph RBD in production, so the StatefulSet manifest you write here works unchanged on a real cluster.
>
> 💡 Замечание про кластер из лабы 12 по-прежнему в силе: k3d поставляет `local-path` (`rancher.io/local-path`) как `StorageClass` по умолчанию, помеченный `(default)`. Он поддерживает только RWO, динамически провижинится через hostPath и идеально подходит для счётчика на три реплики; при этом его *интерфейс* идентичен EBS / GCE PD / Ceph RBD в продакшене, поэтому манифест StatefulSet, который ты здесь напишешь, без изменений заработает на реальном кластере.

Directory layout (you'll fill these files yourself):

Структура каталогов (эти файлы ты заполнишь сам):

```
lab10-app/                              # your Helm chart from Lab 10
├── templates/
│   ├── statefulset.yaml                # YOU write this (§Task 2)
│   ├── service-headless.yaml           # YOU write this (§Task 3)
│   ├── service.yaml                    # exists from Lab 9 — keep as-is
│   └── deployment.yaml                 # exists from Lab 9 — keep, gate behind values
└── values.yaml                         # add: kind toggle, replicaCount, persistence.*
k8s/docs/
└── LAB15.md                            # your submission report
```

Пояснение: `statefulset.yaml` и `service-headless.yaml` создаёшь ты; `service.yaml` и `deployment.yaml` остаются из лабы 9, причём Deployment отключается через values; в `values.yaml` добавляются переключатель типа, `replicaCount` и настройки `persistence.*`.

---

## Task 1 — Concepts: When Identity & Per-Pod Storage Matter (2 pts)

Задание 1 — Концепции: когда важны идентичность и хранилище для каждого пода (2 балла)

Before you write a single line of YAML, write down *why* you're writing it. This task is documentation-only — answer in `k8s/docs/LAB15.md` (you'll extend the same file in every subsequent task).

Прежде чем написать хоть одну строку YAML, сформулируй, *зачем* ты её пишешь. Это задание только на документацию — отвечай в `k8s/docs/LAB15.md` (этот же файл ты будешь дополнять в каждом следующем задании).

`YOUR TASK`: in 2–3 sentences each, answer:

`YOUR TASK`: ответь на каждый вопрос в 2–3 предложениях:

1. **The three guarantees.** What does a StatefulSet give you that a Deployment does not? Hint — the slide 5 list is: stable identity, stable storage, ordered lifecycle. Put them in **your own words** with one sentence each on *which kind of workload needs each one and why*.
   * **Три гарантии.** Что даёт StatefulSet, чего не даёт Deployment? Подсказка: список со слайда 5 — стабильная идентичность, стабильное хранилище, упорядоченный жизненный цикл. Сформулируй их **своими словами** и добавь по одному предложению о том, *какому типу нагрузки нужна каждая гарантия и почему*.

2. **Why a Deployment is wrong for Postgres.** Pick **two** of: random pod names, shared (or no) PVC, parallel startup, pod IPs change on restart. For each one, write one sentence on the specific failure mode it causes in a Postgres cluster (lecture 15 slide 3 has the table).
   * **Почему Deployment не подходит для Postgres.** Выбери **два** пункта из: случайные имена подов, общий (или отсутствующий) PVC, параллельный запуск, смена IP пода при перезапуске. Для каждого напиши одно предложение о конкретном сценарии сбоя, который это вызывает в кластере Postgres (таблица на слайде 3 лекции 15).

3. **Headless Services.** Explain in your own words what `clusterIP: None` does to DNS, and why a StatefulSet *requires* a governing headless Service to address individual pods. (Hint: regular Service returns one VIP; headless returns N A records *plus* per-pod records.)
   * **Headless Services.** Объясни своими словами, что `clusterIP: None` делает с DNS и почему StatefulSet *требует* управляющего headless Service для адресации отдельных подов. (Подсказка: обычный Service возвращает один VIP; headless возвращает N A-записей *плюс* записи для каждого пода.)

4. **When NOT to roll your own.** In 2-3 sentences, when would you stop using a raw StatefulSet and reach for an operator (e.g. CloudNativePG, Strimzi, Rook)? Hint: failover, coordinated backups, point-in-time restore.
   * **Когда не писать своё.** В 2–3 предложениях: когда стоит перестать использовать «голый» StatefulSet и взять оператор (например, CloudNativePG, Strimzi, Rook)? Подсказка: фейловер, согласованные бэкапы, восстановление на точку во времени.

`YOUR TASK`: fill in the Deployment vs StatefulSet comparison table in `LAB15.md`. The columns are given; the rows are yours to populate.

`YOUR TASK`: заполни таблицу сравнения Deployment и StatefulSet в `LAB15.md`. Колонки даны; строки заполняешь ты.

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | `YOUR TASK` | `YOUR TASK` |
| Storage | `YOUR TASK` | `YOUR TASK` |
| Scaling order | `YOUR TASK` | `YOUR TASK` |
| Network identity | `YOUR TASK` | `YOUR TASK` |
| Termination order | `YOUR TASK` | `YOUR TASK` |

| Возможность | Deployment | StatefulSet |
|---|---|---|
| Имена подов | `YOUR TASK` | `YOUR TASK` |
| Хранилище | `YOUR TASK` | `YOUR TASK` |
| Порядок масштабирования | `YOUR TASK` | `YOUR TASK` |
| Сетевая идентичность | `YOUR TASK` | `YOUR TASK` |
| Порядок завершения | `YOUR TASK` | `YOUR TASK` |

### Proof of work

Подтверждение работы

**Paste into `k8s/docs/LAB15.md`:**

**Вставь в `k8s/docs/LAB15.md`:**

- All four answers above, in your own words
  * Все четыре ответа выше, своими словами
- The completed comparison table
  * Заполненную таблицу сравнения
- One sentence applying the *operator rule of thumb* to your `visits` app: would *you* run it as a raw StatefulSet, or reach for an operator? Why?
  * Одно предложение с применением *правила большого пальца про операторы* к твоему приложению `visits`: запустил бы *ты* его как «голый» StatefulSet или взял бы оператор? Почему?

---

## Task 2 — Convert the Deployment to a StatefulSet (3 pts)

Задание 2 — Преврати Deployment в StatefulSet (3 балла)

The headline manifest. Below is the **skeleton** — the parts that are essentially the same as your Lab 10 Deployment template are filled in (chart helpers, container shape). The parts that *are the StatefulSet skill* — `replicas`, `selector.matchLabels`, the whole pod template body, and the entire `volumeClaimTemplates` block — are blank. Write them.

Главный манифест. Ниже — **скелет**: части, которые по сути совпадают с шаблоном Deployment из лабы 10 (хелперы чарта, форма контейнера), уже заполнены. Части, которые *и есть навык работы со StatefulSet*, — `replicas`, `selector.matchLabels`, всё тело шаблона пода и весь блок `volumeClaimTemplates` — пусты. Напиши их.

### 2.1 — Author `templates/statefulset.yaml`

2.1 — Напиши `templates/statefulset.yaml`

```yaml
# lab10-app/templates/statefulset.yaml
apiVersion: apps/v1                                     # GA since K8s 1.9 — the only group/version
kind: StatefulSet
metadata:
  name: {{ include "lab10-app.fullname" . }}-web        # same `-web` component suffix as your Lab 10 Deployment;
                                                        # rendered name on the default release will be `lab10-app-web`,
                                                        # pods will be `lab10-app-web-0`, `lab10-app-web-1`, `lab10-app-web-2`.
  labels:
    {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  serviceName: ___                                      # YOUR TASK: must equal the name of your headless Service in Task 3
                                                        #            ⚠️ if this is missing or wrong, the controller
                                                        #            still creates pods but per-pod DNS silently breaks
  replicas: ___                                         # YOUR TASK: how many ordinals do you need to
                                                        #            demonstrate per-pod identity AND prove the
                                                        #            data doesn't cross between pods?
  selector:
    matchLabels:
      ___: ___                                          # YOUR TASK: the label that matches pod template labels
                                                        #            AND the headless Service selector — immutable
  template:
    metadata:
      labels:
        ___: ___                                        # YOUR TASK: MUST equal selector.matchLabels above
    spec:
      containers:
        - name: app                                     # YOUR TASK: write the full container spec —
          # YOUR TASK: image, ports, env (DATA_DIR from Lab 12),
          # YOUR TASK: volumeMounts (data → /data, matching the volumeClaimTemplate name),
          # YOUR TASK: readinessProbe pointing at /health (cheap; not the visit counter).
          # The shape is identical to your Lab 10 Deployment container — copy that, then
          # change the volume name to match the volumeClaimTemplate you'll write below.
  volumeClaimTemplates:
    # YOUR TASK: the whole list. Each entry is a PVC template — it LOOKS like a normal PVC
    #            with metadata + spec, but it lives INSIDE the StatefulSet. The `metadata.name`
    #            becomes the PREFIX of the per-pod claim:  data-<sts>-0,  data-<sts>-1,  data-<sts>-2.
    #            Required fields:
    #              - metadata.name              (e.g. "data" — matches the volumeMount above)
    #              - spec.accessModes           (which one for one-Deployment-replica-style writes?)
    #              - spec.resources.requests.storage  (size — counter is one int)
    #              - spec.storageClassName      (optional; omit = use cluster default = local-path on k3d)
```

Пояснение: в скелете заполнены только каркасные части; поля с `___` и блок `volumeClaimTemplates` нужно написать самому. Комментарии в коде подсказывают, что именно проверяется и какие ошибки типичны.

**Why each blank matters — read before filling:**

**Почему важно каждое пустое место — прочитай перед заполнением:**

- **`spec.serviceName`** — the StatefulSet ↔ DNS bond. The controller uses this name to build the per-pod FQDN `<pod>.<serviceName>.<ns>.svc.cluster.local`. **A missing or wrong `serviceName` is a silent failure: pods still come up, but `kubectl exec web-0 -- getent hosts web-1.<wrong-svc>` returns nothing.** The StatefulSet docs literally call this field out as required.
  * **`spec.serviceName`** — связь StatefulSet ↔ DNS. Контроллер использует это имя, чтобы построить FQDN пода `<pod>.<serviceName>.<ns>.svc.cluster.local`. **Отсутствующий или неверный `serviceName` — тихий отказ: поды всё равно поднимаются, но `kubectl exec web-0 -- getent hosts web-1.<wrong-svc>` ничего не вернёт.** Документация StatefulSet прямо называет это поле обязательным.
- **`replicas`** — pick a number that lets you prove **two** things at once: (a) ordinal identity (need at least 2 to show `web-0 ≠ web-1`), and (b) per-pod PVC binding (the more pods, the more convincing the proof). The reference uses 3.
  * **`replicas`** — выбери число, которое позволяет доказать **две** вещи одновременно: (а) порядковую идентичность (нужно минимум 2, чтобы показать `web-0 ≠ web-1`) и (б) привязку PVC к каждому поду (чем больше подов, тем убедительнее). В эталоне используется 3.
- **The label triangle** — same #1 K8s YAML error as Lab 9. `selector.matchLabels` must equal `template.metadata.labels`. The headless Service's `spec.selector` in Task 3 must match the *same* labels.
  * **Треугольник лейблов** — та же ошибка №1 в YAML для Kubernetes, что и в лабе 9. `selector.matchLabels` должен совпадать с `template.metadata.labels`. `spec.selector` headless Service в задании 3 должен ссылаться на *те же* лейблы.
- **`volumeClaimTemplates`** — this is **not a PVC**, it is a *template* the controller stamps out one PVC per ordinal from. The block is **immutable after creation**. If you need to resize, you edit the PVCs directly (only works if the StorageClass has `allowVolumeExpansion: true`) — re-templating won't propagate. **Don't bury this in your head until later; lecture 15 slide 19 calls it the resizing trap.**
  * **`volumeClaimTemplates`** — это **не PVC**, а *шаблон*, по которому контроллер штампует по одному PVC на каждый порядковый номер. Блок **неизменяем после создания**. Если нужно увеличить размер, редактируй PVC напрямую (сработает, только если у StorageClass есть `allowVolumeExpansion: true`) — повторный рендеринг шаблона изменения не применит. **Не откладывай это на потом; слайд 19 лекции 15 называет это ловушкой изменения размера.**
- **The container spec** — your `volumeMounts[].name` must equal the `volumeClaimTemplates[].metadata.name`. Mis-name them and the pod starts with `emptyDir` and zero error message — the writes just don't persist.
  * **Спецификация контейнера** — `volumeMounts[].name` должен совпадать с `volumeClaimTemplates[].metadata.name`. Если назвать их по-разному, под запустится с `emptyDir` и без единой ошибки — записи просто не сохранятся.

### 2.2 — Update `values.yaml` and gate the old Deployment

2.2 — Обнови `values.yaml` и отключи старый Deployment

Add to `values.yaml`:

Добавь в `values.yaml`:

```yaml
# YOUR TASK: add the keys this lab needs. Required:
#   - a toggle that picks StatefulSet over Deployment (so deployment.yaml doesn't render)
#   - replicaCount (Task 2)
#   - persistence.size, persistence.storageClass (Task 2 volumeClaimTemplates)
# Suggested shape:
# kind: StatefulSet            # toggle: "StatefulSet" | "Deployment"
# replicaCount: ...
# persistence:
#   size: ...
#   storageClass: ""           # "" → cluster default = local-path on k3d
```

Пояснение: нужен переключатель, который выбирает StatefulSet вместо Deployment, число реплик и параметры хранилища — размер и класс. Пустой `storageClass` означает класс по умолчанию, то есть `local-path` в k3d.

`YOUR TASK`: in `templates/deployment.yaml` (your Lab 10 file), wrap the whole thing in `{{- if eq .Values.kind "Deployment" }} ... {{- end }}` so it doesn't render when you flip the toggle. Do the same gate at the top of `statefulset.yaml` with `eq .Values.kind "StatefulSet"`. **Keep both files in-tree** — graders will check that you didn't delete history.

`YOUR TASK`: в `templates/deployment.yaml` (файле из лабы 10) оберни всё содержимое в `{{- if eq .Values.kind "Deployment" }} ... {{- end }}`, чтобы он не рендерился при переключении. Такой же гейт добавь в начало `statefulset.yaml` с условием `eq .Values.kind "StatefulSet"`. **Оба файла оставь в репозитории** — проверяющие будут смотреть, что ты не удалял историю.

### 2.3 — Deploy and verify

2.3 — Задеплой и проверь

```bash
helm upgrade --install lab10-app ./lab10-app -n lab15 --create-namespace
kubectl get statefulset -n lab15                # YOUR TASK: should show 3/3 READY
kubectl get pods -n lab15 -l app.kubernetes.io/name=lab10-app  # YOUR TASK: ordinal pods, NOT random suffixes
kubectl get pvc -n lab15                        # YOUR TASK: 3 PVCs named data-<sts>-0/1/2, all Bound
```

Пояснение: `helm upgrade --install` ставит или обновляет релиз в пространстве имён `lab15`; следующие три команды показывают StatefulSet, поды и PVC — ожидаются три готовые реплики с порядковыми именами и три привязанных тома.

> 💡 **Debug ladder when this doesn't work first time:**
> 1. `kubectl describe statefulset <name>` — if the controller refuses to roll, the Events at the bottom name the field.
> 2. `kubectl get pvc` — `Pending` PVCs with `local-path` is **normal** (`WaitForFirstConsumer`); `Pending` PVCs without a pod consumer = wrong access mode or storage class.
> 3. `kubectl describe pod <name>-0` — bottom Events on a CrashLoopBackOff or never-starts.
> 4. **`spec.selector.matchLabels` is immutable** — if you got it wrong on first apply, `helm uninstall lab10-app -n lab15`, `kubectl delete pvc -n lab15 -l app.kubernetes.io/name=lab10-app`, then `helm upgrade --install` again.
>
> 💡 **Порядок отладки, если с первого раза не заработало:**
> 1. `kubectl describe statefulset <name>` — если контроллер отказывается катить, нужное поле будет названо в Events внизу.
> 2. `kubectl get pvc` — `Pending` у PVC с `local-path` — это **нормально** (`WaitForFirstConsumer`); `Pending` без пода-потребителя означает неверный режим доступа или класс хранилища.
> 3. `kubectl describe pod <name>-0` — смотри Events внизу при CrashLoopBackOff или незапускающемся поде.
> 4. **`spec.selector.matchLabels` неизменяем** — если ошибся при первом apply, выполни `helm uninstall lab10-app -n lab15`, `kubectl delete pvc -n lab15 -l app.kubernetes.io/name=lab10-app`, затем снова `helm upgrade --install`.

### 2.4 — Proof of work

2.4 — Подтверждение работы

**Paste into `k8s/docs/LAB15.md`:**

**Вставь в `k8s/docs/LAB15.md`:**

- `kubectl get sts,pod,pvc -n lab15` — must show **3/3** ready, ordinal pod names, **3 PVCs** named `data-<sts>-0/1/2`, all `Bound`
  * `kubectl get sts,pod,pvc -n lab15` — должно показать **3/3** готовых, порядковые имена подов, **3 PVC** с именами `data-<sts>-0/1/2`, все в статусе `Bound`
- One sentence on which `accessModes` you chose for `volumeClaimTemplates` and why (`RWO` is correct here — defend it; `RWX` is wrong on local-path)
  * Одно предложение о том, какой `accessModes` ты выбрал для `volumeClaimTemplates` и почему (`RWO` здесь правильный — обоснуй; `RWX` неверен для local-path)
- One sentence on **why your `serviceName` value matches the headless Service name** in Task 3 (and what would break if it didn't)
  * Одно предложение о том, **почему значение `serviceName` совпадает с именем headless Service** из задания 3 (и что сломалось бы, если бы не совпадало)
- The relevant slice of your `statefulset.yaml` — the `volumeClaimTemplates` block specifically, so the grader can see how you defined the per-pod PVC
  * Нужный фрагмент твоего `statefulset.yaml` — конкретно блок `volumeClaimTemplates`, чтобы проверяющий видел, как ты определил PVC для каждого пода

---

## Task 3 — Headless Service & Per-Pod DNS (2 pts)

Задание 3 — Headless Service и DNS для отдельных подов (2 балла)

This is where `clusterIP: None` earns its keep. A regular Service gets a virtual IP (one entry, kube-proxy load-balances). A **headless** Service has no IP at all — DNS returns the A records of *every* matching pod, *plus* per-pod records of the form `<pod>.<service>.<ns>.svc.cluster.local`. That second form is what makes pod-0 directly addressable in a way pod-1 can't impersonate.

Здесь `clusterIP: None` оправдывает своё существование. Обычный Service получает виртуальный IP (одна запись, kube-proxy балансирует). У **headless** Service нет IP вообще — DNS возвращает A-записи *каждого* подходящего пода *плюс* записи вида `<pod>.<service>.<ns>.svc.cluster.local`. Вторая форма и делает pod-0 напрямую адресуемым так, что pod-1 не может выдать себя за него.

### 3.1 — Author `templates/service-headless.yaml`

3.1 — Напиши `templates/service-headless.yaml`

The skeleton below shows only the *kind* and `apiVersion` — every other field is yours.

В скелете ниже показаны только *kind* и `apiVersion` — все остальные поля твои.

```yaml
# lab10-app/templates/service-headless.yaml
apiVersion: v1                                          # Services live in core v1
kind: Service
metadata:
  name: ___                                             # YOUR TASK: must equal spec.serviceName from your StatefulSet
                                                        #            (Task 2). Pick a name that makes the per-pod FQDN
                                                        #            readable, e.g. <chart>-headless.
  labels:
    {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  clusterIP: ___                                        # YOUR TASK: ONE specific value — the field is what makes this
                                                        #            Service "headless". A typo (e.g. an empty string,
                                                        #            or omitting the field) gives you a regular VIP
                                                        #            and silently defeats per-pod DNS.
  selector:
    ___: ___                                            # YOUR TASK: must match the StatefulSet pod template labels
                                                        #            EXACTLY — same label triangle as Task 2.
  ports:
    - ___                                               # YOUR TASK: the full port list — `name`, `port`, `targetPort`.
                                                        #            One entry pointing at your app's HTTP port. The
                                                        #            `port` field on a headless Service is informational
                                                        #            (no kube-proxy is involved), but you still need it
                                                        #            for DNS SRV records and for kubectl's port column.
```

Пояснение: как и в предыдущем скелете, нужно самостоятельно заполнить имя, `clusterIP`, селектор и порты; комментарии в коде перечисляют, что именно проверяется.

**Why each blank matters:**

**Почему важно каждое пустое место:**

- **`metadata.name`** — this is the second half of the FQDN. Get it wrong and `serviceName` in the StatefulSet points at nothing → per-pod DNS broken, with no error from the StatefulSet controller (it doesn't validate that the Service exists).
  * **`metadata.name`** — это вторая половина FQDN. Ошибёшься — и `serviceName` в StatefulSet будет указывать в никуда → DNS для отдельных подов сломается, причём контроллер StatefulSet не выдаст ошибку (он не проверяет, существует ли Service).
- **`spec.clusterIP`** — the single field that distinguishes headless from regular. Spelled wrong / set to `""` / set to `192.x.x.x` and you get either a Service that won't apply or one that has a VIP and load-balances — which silently breaks the *whole point* of this lab. **There is exactly one valid value.**
  * **`spec.clusterIP`** — единственное поле, отличающее headless от обычного. Опечатка / `""` / `192.x.x.x` — и получится либо Service, который не применится, либо обычный с VIP и балансировкой, что тихо ломает *весь смысл* этой лабы. **Существует ровно одно допустимое значение.**
- **`spec.selector`** — the third corner of the label triangle. `kubectl get endpoints <headless-svc>` is your debugging oracle: it must list the pod IPs of your StatefulSet pods. Empty list = selector typo.
  * **`spec.selector`** — третий угол треугольника лейблов. `kubectl get endpoints <headless-svc>` — твой оракул отладки: он должен перечислять IP подов твоего StatefulSet. Пустой список = опечатка в селекторе.
- **`spec.ports`** — you still declare them. Headless Services don't load-balance, but kube-DNS uses the port name for SRV records (cluster-internal service discovery) and `kubectl get svc` shows `<none>` for ClusterIP only if you also forget the port.
  * **`spec.ports`** — их всё равно нужно объявлять. Headless Service не балансирует нагрузку, но kube-DNS использует имя порта для SRV-записей (service discovery внутри кластера), а `kubectl get svc` покажет `<none>` в колонке ClusterIP, только если ты ещё и забудешь порт.

### 3.2 — Apply and verify per-pod DNS

3.2 — Примени и проверь DNS для отдельных подов

```bash
kubectl apply ...                                                # the `helm upgrade` from Task 2 already deployed it
kubectl get svc -n lab15                                         # YOUR TASK: confirm CLUSTER-IP column shows "None"
                                                                 #            for your headless Service
kubectl get endpoints <your-headless-svc> -n lab15               # YOUR TASK: MUST list 3 pod IPs (one per replica)
```

Пояснение: `kubectl get svc` должен показать `None` в колонке CLUSTER-IP у headless Service, а `kubectl get endpoints` — три IP подов, по одному на реплику.

`YOUR TASK`: prove that the per-pod DNS record actually resolves. **busybox / your slim Lab 2 image may not ship `nslookup` or `dig`** — use one of these instead:

`YOUR TASK`: докажи, что DNS-запись для отдельного пода действительно резолвится. **В busybox / твоём slim-образе из лабы 2 может не быть `nslookup` или `dig`** — используй один из вариантов ниже:

```bash
# Option A — getent (in glibc; works in most images, fails in busybox/alpine without bind-tools)
kubectl exec -n lab15 <pod-0> -- getent hosts <pod-1>.<headless>.lab15.svc.cluster.local

# Option B — throwaway debug pod with full networking tools
kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -n lab15 -- \
  nslookup <pod-1>.<headless>.lab15.svc.cluster.local
```

Пояснение: вариант A использует `getent`, если в образе есть glibc; вариант B поднимает временный отладочный под `nicolaka/netshoot` с полным набором сетевых утилит.

Expect: a single A record matching `<pod-1>`'s current IP, with the FQDN pattern `<pod>.<headless>.<ns>.svc.cluster.local`. The name never changes for a given ordinal even if pod-1 is rescheduled to another node.

Ожидается: одна A-запись с текущим IP `<pod-1>` и FQDN вида `<pod>.<headless>.<ns>.svc.cluster.local`. Для заданного порядкового номера имя не меняется, даже если pod-1 переедет на другой узел.

### 3.3 — Observe ordered startup

3.3 — Понаблюдай за упорядоченным запуском

```bash
kubectl get pods -n lab15 -w                       # in one terminal — watch ordinal-by-ordinal startup
kubectl scale statefulset <name> -n lab15 --replicas=1   # in another — terminates -2 then -1 (reverse)
kubectl scale statefulset <name> -n lab15 --replicas=3   # back up — brings -1 then -2 in order
```

Пояснение: первая команда в одном терминале показывает запуск подов по порядку; вторая в другом терминале уменьшает число реплик до одной — первыми завершатся pod-2, затем pod-1; третья возвращает три реплики, и они поднимаются в порядке pod-1, затем pod-2.

`YOUR TASK`: in `LAB15.md`, write 2–3 sentences on the default `OrderedReady` behaviour you observed, and one example of when `podManagementPolicy: Parallel` is appropriate (hint: Cassandra-style gossip clusters where every node is equal — most databases want the default).

`YOUR TASK`: в `LAB15.md` напиши 2–3 предложения о поведении `OrderedReady` по умолчанию, которое ты наблюдал, и один пример, когда уместен `podManagementPolicy: Parallel` (подсказка: кластеры на gossip-протоколе вроде Cassandra, где все узлы равны; большинству баз данных нужен режим по умолчанию).

### 3.4 — Proof of work

3.4 — Подтверждение работы

**Paste into `k8s/docs/LAB15.md`:**

**Вставь в `k8s/docs/LAB15.md`:**

- `kubectl get svc -n lab15` showing the headless Service with `CLUSTER-IP: None`, alongside your existing regular Service (so the grader sees both coexist)
  * `kubectl get svc -n lab15` с headless Service и `CLUSTER-IP: None` рядом с существующим обычным Service (чтобы проверяющий увидел, что оба сосуществуют)
- `kubectl get endpoints <headless> -n lab15` showing **3** pod IPs
  * `kubectl get endpoints <headless> -n lab15` с **тремя** IP подов
- The per-pod DNS resolution output from §3.2, with the FQDN pattern called out
  * Вывод резолвинга DNS для отдельного пода из §3.2 с явно указанным шаблоном FQDN
- The watched-pods capture from §3.3 (paste the relevant lines — ordinal 0 Ready before 1 starts; on scale-down, 2 terminates before 1)
  * Запись наблюдения за подами из §3.3 (вставь нужные строки — под с порядковым номером 0 становится Ready до запуска 1; при уменьшении числа реплик pod-2 завершается раньше pod-1)
- Your 2–3 sentences on `OrderedReady` vs `Parallel`
  * Твои 2–3 предложения о `OrderedReady` и `Parallel`

---

## Task 4 — Per-Pod PVC Persistence Proof (2 pts) ← **headline task**

Задание 4 — Доказательство сохранности PVC для каждого пода (2 балла) ← **главное задание**

This is the whole point of the lab. **A StatefulSet that runs is not the proof. The proof is:** write a unique marker into `web-1`'s volume, delete `web-1`, wait for the controller to recreate it, read the marker back from the **new** `web-1` — same value. Then write a *different* marker into `web-0`'s volume and show that the markers don't cross — `web-0`'s value is its own, not `web-1`'s.

В этом и состоит весь смысл лабы. **Работающий StatefulSet — ещё не доказательство. Доказательство — это:** записать уникальный маркер в том `web-1`, удалить `web-1`, дождаться, пока контроллер пересоздаст его, прочитать маркер из **нового** `web-1` — значение то же. Затем записать *другой* маркер в том `web-0` и показать, что маркеры не пересекаются: значение у `web-0` своё, а не от `web-1`.

### 4.1 — Write a per-pod marker

4.1 — Запиши маркер для отдельного пода

`YOUR TASK`: figure out the four steps yourself. Each one is a `kubectl exec` or `kubectl delete` — no manifest changes.

`YOUR TASK`: сам придумай четыре шага. Каждый — это `kubectl exec` или `kubectl delete`; манифесты менять не нужно.

```bash
# YOUR TASK: write a UNIQUE marker (epoch seconds, your name, anything distinctive) into
#            web-1's mounted volume — into /data/marker, on web-1 only
#
# YOUR TASK: kubectl delete pod <sts>-1 -n lab15
#            (the StatefulSet controller will immediately reschedule it — same name,
#             possibly different node, same PVC reattached)
#
# YOUR TASK: kubectl wait --for=condition=Ready pod/<sts>-1 -n lab15 --timeout=60s
#            (so the next exec doesn't race the pod's startup)
#
# YOUR TASK: read /data/marker from the NEW <sts>-1 — assert it MATCHES the value you wrote
```

Пояснение: шаги — записать маркер в `/data/marker` на `web-1`, удалить под, дождаться его готовности и прочитать файл из нового пода. Если значение совпало, привязка PVC к порядковому номеру доказана.

If the marker comes back the **same**, you've proven the headline guarantee: `data-<sts>-1` is bound to ordinal 1, not to the pod instance. The new pod inherited the old pod's PVC.

Если маркер вернулся **тем же**, ты доказал главную гарантию: `data-<sts>-1` привязан к порядковому номеру 1, а не к экземпляру пода. Новый под унаследовал PVC старого.

If the marker is **missing** (file not found) → your `volumeClaimTemplates` didn't mount at `DATA_DIR`, or you wrote to an `emptyDir` by accident (volume name mismatch — see Task 2 hints).

Если маркер **пропал** (файл не найден) → `volumeClaimTemplates` не смонтировался в `DATA_DIR`, или ты случайно писал в `emptyDir` (несовпадение имени тома — см. подсказки в задании 2).

If the marker comes back **different / empty** → you're not using `volumeClaimTemplates`; you're using a shared PVC mounted twice, which is wrong for RWO.

Если маркер вернулся **другим / пустым** → ты используешь не `volumeClaimTemplates`, а общий PVC, смонтированный дважды, что неверно для RWO.

### 4.2 — Prove the markers don't cross

4.2 — Докажи, что маркеры не пересекаются

`YOUR TASK`: write a **different** marker into `web-0`'s volume. Then read both back. Show that `web-0` has its own marker and `web-1` has its own — they are not the same. (This is the per-pod *isolation* half of the proof. Without it, you've only shown one pod's PVC survived, not that pod-N owns its own PVC.)

`YOUR TASK`: запиши **другой** маркер в том `web-0`. Затем прочитай оба. Покажи, что у `web-0` свой маркер, а у `web-1` свой — они разные. (Это половина доказательства про *изоляцию* подов. Без неё ты показал только, что PVC одного пода выжил, но не что pod-N владеет собственным PVC.)

```bash
# YOUR TASK: write a DIFFERENT marker into <sts>-0 's /data/marker
# YOUR TASK: read /data/marker from <sts>-0 AND from <sts>-1
#            They must be DIFFERENT — each pod's marker is its own.
```

Пояснение: маркеры в `/data/marker` у `web-0` и `web-1` должны быть разными — это доказывает, что у каждого пода отдельный том, а не общий.

### 4.3 — One more sanity check — the visits counter

4.3 — Ещё одна проверка — счётчик посещений

Your Lab 12 `/visits` counter now lives on a per-pod PVC. Drive traffic to each pod individually and show that the counters are *independent*:

Твой счётчик `/visits` из лабы 12 теперь живёт на PVC каждого отдельного пода. Погоняй трафик к каждому поду по отдельности и покажи, что счётчики *независимы*:

```bash
kubectl port-forward -n lab15 pod/<sts>-0 8080:8080 &
kubectl port-forward -n lab15 pod/<sts>-1 8081:8080 &
# YOUR TASK: curl localhost:8080/visits   — pod-0's counter only
# YOUR TASK: curl localhost:8081/visits   — pod-1's counter only
# YOUR TASK: hit pod-0 a few extra times; pod-1's count must NOT change
```

Пояснение: два `port-forward` дают доступ к портам 8080 и 8081 каждого пода; запросы к одному поду не должны менять счётчик другого.

### 4.4 — Proof of work

4.4 — Подтверждение работы

**Paste into `k8s/docs/LAB15.md` (this is the headline section — make it impossible to miss):**

**Вставь в `k8s/docs/LAB15.md` (это главный раздел — сделай так, чтобы его невозможно было пропустить):**

- **The §4.1 capture**, side by side: `kubectl exec <sts>-1 -- cat /data/marker` (original) → `kubectl delete pod <sts>-1` → `kubectl wait ...` → `kubectl exec <sts>-1 -- cat /data/marker` (new pod, **same value**)
  * **Запись из §4.1**, бок о бок: `kubectl exec <sts>-1 -- cat /data/marker` (исходный под) → `kubectl delete pod <sts>-1` → `kubectl wait ...` → `kubectl exec <sts>-1 -- cat /data/marker` (новый под, **то же значение**)
- **The §4.2 capture**: both pods' markers, clearly different values
  * **Запись из §4.2**: маркеры обоих подов с явно разными значениями
- The §4.3 visit counter divergence — 2-3 curl outputs proving pod-0's count is decoupled from pod-1's
  * Расхождение счётчиков из §4.3 — 2–3 вывода `curl`, доказывающие, что счётчик pod-0 не связан со счётчиком pod-1
- One sentence on **why** this works (PVC bound to ordinal via `volumeClaimTemplates`, controller reattaches on reschedule)
  * Одно предложение о том, **почему** это работает (PVC привязан к порядковому номеру через `volumeClaimTemplates`, контроллер переподключает его при пересоздании)

---

## Task 5 — Documentation (1 pt)

Задание 5 — Документация (1 балл)

Pull everything together into `k8s/docs/LAB15.md`. Required sections, in order:

Собери всё в `k8s/docs/LAB15.md`. Обязательные разделы по порядку:

1. **Concepts** — the four answers and the comparison table from Task 1
   * **Концепции** — четыре ответа и таблица сравнения из задания 1
2. **Manifests** — short paragraphs on the choices you made in your `statefulset.yaml` and `service-headless.yaml`: which `accessModes`, which storage class, which `serviceName`, which `clusterIP`. Cite the *specific failure each would have caused* if you'd picked wrong.
   * **Манифесты** — короткие абзацы о выборе в твоих `statefulset.yaml` и `service-headless.yaml`: какие `accessModes`, какой класс хранилища, какой `serviceName`, какой `clusterIP`. Укажи *конкретный сбой, который вызвал бы каждый неверный выбор*.
3. **Resource inventory** — `kubectl get sts,pod,pvc,svc -n lab15` showing the StatefulSet, ordinal pods, per-ordinal PVCs, both Services (regular + headless)
   * **Инвентарь ресурсов** — `kubectl get sts,pod,pvc,svc -n lab15` со StatefulSet, порядковыми подами, PVC на каждый порядковый номер и обоими Service (обычным и headless)
4. **Network identity** — the per-pod DNS resolution from Task 3.2 with the FQDN pattern explained, plus the ordered startup/scale-down observation
   * **Сетевая идентичность** — резолвинг DNS для отдельного пода из задания 3.2 с объяснением шаблона FQDN, а также наблюдение за упорядоченным запуском и масштабированием вниз
5. **Per-pod PVC persistence proof** — the Task 4 captures: write-delete-wait-read on `web-1` with the same marker, plus the `web-0` vs `web-1` non-crossing check, plus the visits counter divergence
   * **Доказательство сохранности PVC для каждого пода** — записи из задания 4: запись-удаление-ожидание-чтение на `web-1` с тем же маркером, проверка несовпадения `web-0` и `web-1`, расхождение счётчиков посещений
6. **Common pitfalls you hit** — at least one real one with how you debugged it (a `kubectl describe` line that unblocked you, a missing field, a typo)
   * **Частые ловушки, на которые ты наткнулся** — как минимум одна реальная с описанием отладки (строка из `kubectl describe`, которая помогла, пропущенное поле, опечатка)

> Keep all `kubectl`/`curl` output clearly marked as captured from your cluster. Do not paste output you didn't actually generate.
>
> Помечай весь вывод `kubectl`/`curl` как снятый с твоего кластера. Не вставляй вывод, который ты на самом деле не получал.

---

## Bonus Task — Update Strategies for Stateful Workloads (2 pts)

Бонусное задание — стратегии обновления для stateful-нагрузок (2 балла)

Upgrading the primary of a stateful cluster matters *far* more than swapping a stateless frontend — the cost of an automated mistake is "restore from backup". K8s gives you two strategies (`RollingUpdate` with `partition`, and `OnDelete`), and the broader ecosystem gives you a third (operators that handle upgrades for you).

Обновление первичного узла stateful-кластера значит *гораздо* больше, чем замена stateless-фронтенда: цена автоматической ошибки — «восстанавливай из бэкапа». Kubernetes даёт две стратегии (`RollingUpdate` с `partition` и `OnDelete`), а экосистема — третью (операторы, которые сами управляют обновлениями).

**The problem:** you need to canary a stateful workload — upgrade *one* pod first, validate, then roll forward. A naive `RollingUpdate` of a Deployment ships the new image to every replica with no human gate.

**Проблема:** нужно выкатить stateful-нагрузку канареечно — сначала обновить *один* под, проверить, затем катить дальше. Наивный `RollingUpdate` для Deployment отправляет новый образ на каждую реплику без человеческой проверки.

**The task:** pick **one** of three paths below and implement it. Each is worth the full 2 points. Don't ship YAML you copied from a tutorial — research the strategy, decide why you picked it over the others, document the trade-off.

**Задача:** выбери **один** из трёх путей ниже и реализуй его. Каждый стоит полные 2 балла. Не сдавай YAML, скопированный из урока, — изучи стратегию, реши, почему выбрал её, и опиши компромисс.

### Path A — `RollingUpdate` with `partition`

Путь A — `RollingUpdate` с `partition`

The `partition` field is the StatefulSet equivalent of a canary. Pods with ordinal `>= partition` get the new pod template; the rest stay on the old one. You decrement `partition` to roll the update forward.

Поле `partition` — это аналог канарейки для StatefulSet. Поды с порядковым номером `>= partition` получают новый шаблон пода, остальные остаются на старом. Чтобы продвинуть обновление, уменьшай `partition`.

`YOUR TASK`:

`YOUR TASK`: выполни шаги ниже.

1. Add `spec.updateStrategy` to your StatefulSet. Pick `type: RollingUpdate` with `rollingUpdate.partition` set to your **highest ordinal** so only the top pod updates when you change the image.
   * Добавь `spec.updateStrategy` в свой StatefulSet. Выбери `type: RollingUpdate` с `rollingUpdate.partition`, равным **наибольшему порядковому номеру**, чтобы при смене образа обновлялся только последний под.
2. Change `image.tag` in `values.yaml` to a new version (rebuild your Lab 2 image with a fresh tag, or temporarily change the container `name:` — anything that makes the pod template hash different).
   * Измени `image.tag` в `values.yaml` на новую версию (пересобери образ из лабы 2 с новым тегом или временно измени имя контейнера `name:` — что угодно, лишь бы хеш шаблона пода стал другим).
3. `helm upgrade ...`. Verify that *only* the highest-ordinal pod restarts on the new template. The others stay on the old one.
   * `helm upgrade ...`. Проверь, что перезапускается *только* под с наибольшим порядковым номером на новом шаблоне. Остальные остаются на старом.
4. Walk `partition` down (highest → ... → 0) re-applying each time, and capture the rollout progress between each step.
   * Уменьшай `partition` (от максимума → ... → 0), каждый раз переприменяя изменения, и фиксируй прогресс раската между шагами.

You write the YAML. No skeleton.

YAML пишешь ты. Скелета нет.

### Path B — `OnDelete`

Путь B — `OnDelete`

`type: OnDelete` is the "the controller does *nothing* on a spec change; humans drive the rollout" mode — what most production database upgrades actually look like.

`type: OnDelete` — режим «контроллер *ничего не делает* при изменении спецификации; раскатом управляют люди». Именно так обычно выглядит обновление продакшен-баз данных.

`YOUR TASK`:

`YOUR TASK`: выполни шаги ниже.

1. Add `spec.updateStrategy.type: OnDelete` to your StatefulSet.
   * Добавь `spec.updateStrategy.type: OnDelete` в свой StatefulSet.
2. Change `image.tag` and `helm upgrade ...`. Verify that **no pods restart**. The controller is intentionally inert.
   * Измени `image.tag` и выполни `helm upgrade ...`. Убедись, что **ни один под не перезапускается**. Контроллер намеренно бездействует.
3. `kubectl delete pod <sts>-2` (or whichever ordinal you want to update first). Show that *only that pod* comes back on the new template. The others remain on the old one until you delete them too.
   * Выполни `kubectl delete pod <sts>-2` (или любой другой порядковый номер, который хочешь обновить первым). Покажи, что *только этот под* возвращается на новом шаблоне. Остальные остаются на старом, пока ты их тоже не удалишь.
4. Document one real-world scenario where `OnDelete` is the right answer (hint: strict change windows; coordinated DB primary/replica upgrades where a human checks replication lag between each step).
   * Опиши один реальный сценарий, где `OnDelete` — правильный выбор (подсказка: строгие окна изменений; согласованные обновления primary/replica базы данных, когда человек между шагами проверяет отставание репликации).

You write the YAML. No skeleton.

YAML пишешь ты. Скелета нет.

### Path C — A database operator

Путь C — оператор базы данных

Drop the hand-rolled StatefulSet for one stateful workload and run it via a purpose-built operator instead. Document what the operator does that your raw StatefulSet could not.

Откажись от самописного StatefulSet для одной stateful-нагрузки и запусти её через специально созданный оператор. Опиши, что оператор делает такого, чего не мог твой «голый» StatefulSet.

`YOUR TASK`:

`YOUR TASK`: выполни шаги ниже.

1. Install **one** operator: **CloudNativePG** (PostgreSQL), **Strimzi** (Kafka), or a storage operator like **Rook** (Ceph) / **Longhorn**.
   * Установи **один** оператор: **CloudNativePG** (PostgreSQL), **Strimzi** (Kafka) или оператор хранилища вроде **Rook** (Ceph) / **Longhorn**.
2. Declare a Custom Resource (e.g. CloudNativePG `kind: Cluster` with `instances: 3`).
   * Объяви Custom Resource (например, `kind: Cluster` CloudNativePG с `instances: 3`).
3. Observe that the operator *itself* creates the underlying StatefulSet + Services + PVCs — `kubectl get sts,svc,pvc -n <op-ns>` after applying the CR.
   * Убедись, что оператор *сам* создаёт нижележащие StatefulSet + Services + PVC — `kubectl get sts,svc,pvc -n <op-ns>` после применения CR.
4. Document the value-add in `LAB15.md`: automatic primary election + failover, continuous WAL or topic backups, coordinated rolling upgrades, bundled `PodMonitor` metrics — none of which a raw StatefulSet provides.
   * Опиши в `LAB15.md` дополнительные возможности: автоматические выборы primary и фейловер, непрерывный бэкап WAL или топиков, согласованные rolling-обновления, встроенные метрики `PodMonitor` — ничего из этого «голый» StatefulSet не даёт.

### Proof of work (whichever path you picked)

Подтверждение работы (для выбранного пути)

**Paste into `k8s/docs/LAB15.md`:**

**Вставь в `k8s/docs/LAB15.md`:**

- The updated manifest snippet (your `updateStrategy` block, or your CR YAML for Path C)
  * Обновлённый фрагмент манифеста (твой блок `updateStrategy` или YAML CR для пути C)
- The `kubectl rollout status` / `kubectl get pods -w` capture showing the partial rollout (Paths A & B), or the operator-managed `kubectl get sts,svc,pvc` (Path C)
  * Запись `kubectl rollout status` / `kubectl get pods -w`, показывающую частичный раскат (пути A и B), либо выданный оператором `kubectl get sts,svc,pvc` (путь C)
- **One paragraph** on *why* you picked this path over the other two — the trade-offs, what your `visits` app actually needs, what production tooling you would consider next
  * **Один абзац** о том, *почему* ты выбрал этот путь, а не два других: компромиссы, что на самом деле нужно твоему приложению `visits`, какой продакшен-инструмент ты рассмотрел бы следующим

---

## How to Submit

Как сдавать

```bash
git switch -c lab15
git add lab10-app/ k8s/docs/LAB15.md
git commit -m "feat(lab15): convert chart to StatefulSet with per-pod PVCs + headless Service"
git push -u origin lab15
```

Пояснение: создаётся ветка `lab15`, в коммит добавляются чарт и отчёт, затем ветка отправляется в твой форк.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab15` → `course-repo:master` *(reviewed)*
  * `your-fork:lab15` → `course-repo:master` *(с ревью)*
- `your-fork:lab15` → `your-fork:master` *(merges into your own main)*
  * `your-fork:lab15` → `your-fork:master` *(вливается в твой собственный main)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — concepts, comparison table, operator rule-of-thumb in LAB15.md
- [ ] Task 2 done — StatefulSet with serviceName + replicas + label triangle + volumeClaimTemplates, 3 ordinal pods + 3 Bound per-pod PVCs
- [ ] Task 3 done — headless Service with clusterIP: None, per-pod DNS resolution captured, ordered startup observed
- [ ] Task 4 done — write→delete-pod→read SAME marker on web-1; web-0 marker is its own; visits counters independent per pod
- [ ] Task 5 done — LAB15.md with all 6 sections + real evidence
- [ ] Bonus done — Path A (partition) OR Path B (OnDelete) OR Path C (operator) with justification
```

Пояснение: чек-лист перечисляет ожидаемые результаты по заданиям 1–5 и бонусу; отметь пункты, когда они действительно выполнены.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Concepts (2 pts)

Задание 1 — Концепции (2 балла)

- ✅ Three StatefulSet guarantees described in your own words
  * ✅ Три гарантии StatefulSet описаны своими словами
- ✅ Two specific Deployment failure modes for Postgres explained
  * ✅ Объяснены два конкретных режима отказа Deployment для Postgres
- ✅ Headless Service (`clusterIP: None`) DNS behaviour explained
  * ✅ Объяснено поведение DNS у headless Service (`clusterIP: None`)
- ✅ "When to use an operator" note included
  * ✅ Есть заметка «когда использовать оператор»
- ✅ Comparison table filled in (all five rows)
  * ✅ Заполнена таблица сравнения (все пять строк)

### Task 2 — Convert to StatefulSet (3 pts)

Задание 2 — Превращение в StatefulSet (3 балла)

- ✅ `templates/statefulset.yaml` written by hand
  * ✅ `templates/statefulset.yaml` написан вручную
- ✅ `spec.serviceName` set and matches the headless Service name from Task 3
  * ✅ `spec.serviceName` задан и совпадает с именем headless Service из задания 3
- ✅ Label triangle consistent (selector.matchLabels = template labels = headless Service selector)
  * ✅ Треугольник лейблов согласован (`selector.matchLabels` = лейблы шаблона = селектор headless Service)
- ✅ `replicas: 3` (or justified higher); pods named `<name>-0/1/2`
  * ✅ `replicas: 3` (или больше с обоснованием); поды названы `<name>-0/1/2`
- ✅ `volumeClaimTemplates` block written; per-pod PVCs render as `data-<name>-N` and **all three Bound**
  * ✅ Блок `volumeClaimTemplates` написан; PVC для каждого пода рендерятся как `data-<name>-N`, и **все три в статусе Bound**
- ✅ Container `volumeMounts[].name` matches `volumeClaimTemplates[].metadata.name`
  * ✅ `volumeMounts[].name` контейнера совпадает с `volumeClaimTemplates[].metadata.name`
- ✅ Old `deployment.yaml` gated behind values, not deleted
  * ✅ Старый `deployment.yaml` отключён через values, но не удалён

### Task 3 — Headless Service & Lifecycle (2 pts)

Задание 3 — Headless Service и жизненный цикл (2 балла)

- ✅ `templates/service-headless.yaml` written by hand
  * ✅ `templates/service-headless.yaml` написан вручную
- ✅ `spec.clusterIP: None` set (not omitted, not "")
  * ✅ `spec.clusterIP: None` задан (не пропущен и не `""`)
- ✅ Selector matches the StatefulSet pod labels exactly
  * ✅ Селектор точно совпадает с лейблами подов StatefulSet
- ✅ Regular Service (Lab 9) coexists, not deleted
  * ✅ Обычный Service (лаба 9) сохранён и сосуществует
- ✅ Per-pod DNS resolution captured with FQDN pattern visible
  * ✅ Зафиксирован резолвинг DNS для отдельного пода с видимым шаблоном FQDN
- ✅ Ordered startup / reverse-order scale-down observed
  * ✅ Наблюдается упорядоченный запуск и обратный порядок при масштабировании вниз

### Task 4 — PVC persistence proof (2 pts)

Задание 4 — Доказательство сохранности PVC (2 балла)

- ✅ **`web-1` marker survives `kubectl delete pod web-1`** — same value before and after, captured side by side
  * ✅ **Маркер `web-1` переживает `kubectl delete pod web-1`** — то же значение до и после, зафиксировано бок о бок
- ✅ **`web-0`'s marker is its own** — different from `web-1`'s
  * ✅ **Маркер `web-0` — его собственный** — отличается от маркера `web-1`
- ✅ Visit counters independent per pod
  * ✅ Счётчики посещений независимы у каждого пода
- ✅ One-sentence "why this works" referencing `volumeClaimTemplates` + ordinal binding
  * ✅ Одно предложение «почему это работает» со ссылкой на `volumeClaimTemplates` и привязку к порядковому номеру

### Task 5 — docs (1 pt)

Задание 5 — документация (1 балл)

- ✅ All 6 sections in `k8s/docs/LAB15.md`; real CLI captures (not illustrative)
  * ✅ Все 6 разделов в `k8s/docs/LAB15.md`; реальные записи CLI, а не иллюстративные
- ✅ Pitfalls section includes at least one real one you hit
  * ✅ В разделе ловушек есть хотя бы одна реальная, на которую ты наткнулся

### Bonus — Update Strategies / Operator (2 pts)

Бонус — стратегии обновления / оператор (2 балла)

- ✅ Exactly one path implemented (A / B / C)
  * ✅ Реализован ровно один путь (A / B / C)
- ✅ YAML written by you, not copy-pasted from a tutorial
  * ✅ YAML написан тобой, а не скопирован из урока
- ✅ Proof captures match the path (partition decrement, manual delete, or operator-managed resource list)
  * ✅ Записи-доказательства соответствуют пути (уменьшение `partition`, ручное удаление или список ресурсов, управляемых оператором)
- ✅ Paragraph on *why this path over the other two*
  * ✅ Абзац о том, *почему этот путь, а не два других*

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Concepts | **2** | Guarantees, comparison table, headless DNS, operator rationale |
| **Task 2** — StatefulSet | **3** | Hand-written `statefulset.yaml` with serviceName + label triangle + volumeClaimTemplates; 3 ordinal pods + 3 Bound PVCs |
| **Task 3** — Headless Service & lifecycle | **2** | Hand-written headless Service (`clusterIP: None`); per-pod DNS proven; ordered start observed |
| **Task 4** — Persistence proof | **2** | `web-1` marker survives delete; `web-0` ≠ `web-1`; per-pod isolation proven |
| **Task 5** — Documentation | **1** | Six sections in `LAB15.md` with real evidence |
| **Bonus** — Update strategies / operator | **2** | One of three paths implemented, defended in writing |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Концепции | **2** | Гарантии, таблица сравнения, headless DNS, обоснование оператора |
| **Задание 2** — StatefulSet | **3** | Написанный вручную `statefulset.yaml` с serviceName + треугольником лейблов + volumeClaimTemplates; 3 порядковых пода + 3 привязанных PVC |
| **Задание 3** — Headless Service и жизненный цикл | **2** | Написанный вручную headless Service (`clusterIP: None`); доказан DNS для отдельных подов; наблюдается упорядоченный запуск |
| **Задание 4** — Доказательство сохранности | **2** | Маркер `web-1` переживает удаление; `web-0` ≠ `web-1`; доказана изоляция подов |
| **Задание 5** — Документация | **1** | Шесть разделов в `LAB15.md` с реальными доказательствами |
| **Бонус** — Стратегии обновления / оператор | **2** | Реализован один из трёх путей, обоснован письменно |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading scale (main 10 pts):**

**Шкала оценки (основные 10 баллов):**

- **10/10:** All resources correct, per-pod PVC binding proven on `web-1`, non-crossing demonstrated on `web-0`, documentation excellent
  * **10/10:** Все ресурсы корректны, привязка PVC на каждый под доказана на `web-1`, несовпадение показано на `web-0`, документация отличная
- **8-9/10:** Works end-to-end; minor gaps in evidence or one weak explanation
  * **8–9/10:** Работает от начала до конца; мелкие пробелы в доказательствах или одно слабое объяснение
- **6-7/10:** StatefulSet deploys but persistence proof is thin (only one half — survives delete OR per-pod isolation, not both)
  * **6–7/10:** StatefulSet деплоится, но доказательство сохранности слабое (только половина — переживает удаление ИЛИ изоляция подов, но не оба)
- **<6/10:** Missing headless Service, broken `volumeClaimTemplates`, or no persistence proof
  * **<6/10:** Отсутствует headless Service, сломан `volumeClaimTemplates` или нет доказательства сохранности

---

## Resources

Ресурсы

<details>
<summary>📚 Kubernetes documentation</summary>

📚 Документация Kubernetes

- [StatefulSets — concepts](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
  * [StatefulSets — концепции](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [StatefulSet Basics tutorial](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
  * [Учебник по основам StatefulSet](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/)
- [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
  * [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- [Volume claim templates](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#volume-claim-templates)
  * [Шаблоны запросов на тома (Volume claim templates)](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#volume-claim-templates)
- [Update strategies & partition rollout](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies)
  * [Стратегии обновления и раскат с partition](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  * [Persistent Volumes (постоянные тома)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [DNS for Services and Pods (per-pod records)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pods)
  * [DNS для Service и подов (записи для отдельных подов)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pods)

</details>

<details>
<summary>🤖 Operators & cloud-native storage</summary>

🤖 Операторы и облачное хранилище

- [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) · [OperatorHub.io](https://operatorhub.io/)
  * [Паттерн оператора](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) · [OperatorHub.io](https://operatorhub.io/)
- [CloudNativePG](https://cloudnative-pg.io/) — the PostgreSQL operator (1.29, PG 18.3 default)
  * [CloudNativePG](https://cloudnative-pg.io/) — оператор PostgreSQL (1.29, по умолчанию PG 18.3)
- [Strimzi](https://strimzi.io/) — Kafka on Kubernetes
  * [Strimzi](https://strimzi.io/) — Kafka в Kubernetes
- [Rook](https://rook.io/) — production storage on Ceph (v1.18 + Ceph 19 Squid)
  * [Rook](https://rook.io/) — продакшен-хранилище на Ceph (v1.18 + Ceph 19 Squid)
- [Longhorn](https://longhorn.io/) — lightweight CNCF block storage (1.9)
  * [Longhorn](https://longhorn.io/) — лёгкое блочное хранилище CNCF (1.9)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs — read these BEFORE you debug)</summary>

⚠️ Частые ловушки (из реальных прогонов — прочитай ДО отладки)

- **Missing or wrong `spec.serviceName`.** The StatefulSet controller does **not** fail the apply — it happily creates pods with ordinal names, but per-pod DNS silently breaks because the `<pod>.<serviceName>` FQDN points at no Service. Symptom: pods Ready, but `kubectl exec web-0 -- getent hosts web-1.<wrong>` returns nothing. **Always pair the StatefulSet apply with the matching headless Service apply, and grep both files for the name match before committing.**
  * **Отсутствующий или неверный `spec.serviceName`.** Контроллер StatefulSet **не** проваливает apply — он спокойно создаёт поды с порядковыми именами, но DNS для отдельных подов тихо ломается, потому что FQDN `<pod>.<serviceName>` указывает в никуда. Симптом: поды Ready, но `kubectl exec web-0 -- getent hosts web-1.<wrong>` ничего не возвращает. **Всегда применяй StatefulSet вместе с соответствующим headless Service и перед коммитом проверь совпадение имён в обоих файлах через grep.**

- **`clusterIP: None` typo'd or omitted.** A regular Service with `clusterIP: <some-vip>` has the same shape as a headless one except for that single field. Drop it or set it to `""` and you get a regular VIP that load-balances — which silently defeats per-pod DNS, the whole point of this lab. There is **exactly one valid value**: the literal string `None`.
  * **Опечатка в `clusterIP: None` или его отсутствие.** У обычного Service с `clusterIP: <some-vip>` та же форма, что у headless, кроме этого единственного поля. Убери его или задай `""` — получишь обычный VIP с балансировкой, что тихо убивает DNS для отдельных подов, то есть весь смысл лабораторной. Есть **ровно одно допустимое значение**: литерал `None`.

- **`volumeClaimTemplates` PVCs persist after StatefulSet deletion — the bill-shock trap.** This is deliberate (your data outlives mistakes). It also means: `helm uninstall lab10-app -n lab15` leaves three PVCs sitting around, billing real money on a cloud StorageClass, with no obvious owner. Clean them up explicitly: `kubectl delete pvc -n lab15 -l app.kubernetes.io/name=lab10-app`. **The same trap applies when you fix the immutable `volumeClaimTemplates` block: `helm uninstall` does NOT wipe the PVCs; the next `helm install` may end up with mismatched claims.**
  * **PVC из `volumeClaimTemplates` остаются после удаления StatefulSet — ловушка «шок от счёта».** Это сделано намеренно (данные переживают ошибки). Это также значит, что `helm uninstall lab10-app -n lab15` оставляет три PVC, которые занимают место и стоят реальных денег в облачном StorageClass, без очевидного владельца. Удаляй их явно: `kubectl delete pvc -n lab15 -l app.kubernetes.io/name=lab10-app`. **Та же ловушка срабатывает при исправлении неизменяемого блока `volumeClaimTemplates`: `helm uninstall` НЕ стирает PVC; следующий `helm install` может получить несовпадающие claims.**

- **Ordered pod startup blocks rolling updates when one pod fails liveness.** Default `podManagementPolicy: OrderedReady` means pod-N waits for pod-(N-1) to be Ready before starting. A wedged pod-0 → pods 1, 2, ... never start. Symptom looks like "the StatefulSet is stuck"; root cause is **always** in pod-0's logs/events. Don't reach for `Parallel` to bypass — fix pod-0. Use `Parallel` only when the app's discovery is gossip-based and every node is equal (Cassandra-style).
  * **Упорядоченный запуск подов блокирует rolling-обновления, если один под не проходит liveness.** По умолчанию `podManagementPolicy: OrderedReady` означает, что pod-N ждёт готовности pod-(N-1). Заклинивший pod-0 → поды 1, 2, ... никогда не запустятся. Симптом выглядит как «StatefulSet завис»; первопричина **всегда** в логах и событиях pod-0. Не переключайся на `Parallel`, чтобы обойти проблему, — чини pod-0. Используй `Parallel`, только когда обнаружение в приложении основано на gossip и все узлы равны (как в Cassandra).

- **busybox / slim images lack `nslookup` and `dig`.** Your Lab 2 Python image probably doesn't ship them either. Two clean ways to prove per-pod DNS without bloating the app image:
  * **В busybox / slim-образах нет `nslookup` и `dig`.** В твоём Python-образе из лабы 2 их, скорее всего, тоже нет. Два чистых способа доказать DNS для отдельных подов, не раздувая образ приложения:
  - `kubectl exec <pod> -- getent hosts <fqdn>` — works if the image has glibc (most non-busybox bases do)
    * `kubectl exec <pod> -- getent hosts <fqdn>` — работает, если в образе есть glibc (в большинстве не-busybox баз он есть)
  - `kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -- nslookup <fqdn>` — throwaway debug pod with full tools
    * `kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -- nslookup <fqdn>` — временный отладочный под с полным набором утилит

- **`volumeClaimTemplates` is immutable after creation.** Lecture 15 slide 19 calls this the resizing trap. Edit the block in your chart, `helm upgrade`, and the StatefulSet controller silently ignores the change. To actually resize: `kubectl edit pvc data-<sts>-N` for each ordinal — and only if the StorageClass has `allowVolumeExpansion: true` (the k3d `local-path` default *does* support it; many cloud SCs do too). For deeper changes (storage class, access mode) you must `kubectl delete sts --cascade=orphan` and re-apply with the new template — your PVCs survive the orphan delete.
  * **`volumeClaimTemplates` неизменяем после создания.** Слайд 19 лекции 15 называет это ловушкой изменения размера. Измени блок в чарте, выполни `helm upgrade` — и контроллер StatefulSet молча проигнорирует изменение. Чтобы реально увеличить размер: `kubectl edit pvc data-<sts>-N` для каждого порядкового номера, и только если у StorageClass есть `allowVolumeExpansion: true` (класс `local-path` в k3d по умолчанию *поддерживает* это; многие облачные StorageClass тоже). Для более глубоких изменений (класс хранилища, режим доступа) нужно выполнить `kubectl delete sts --cascade=orphan` и применить заново с новым шаблоном — PVC переживут удаление с orphan.

- **Label triangle drift.** Three places to keep in sync: `StatefulSet.spec.selector.matchLabels`, `StatefulSet.spec.template.metadata.labels`, and `Service.spec.selector` (on both the regular AND headless Services). The selector is **immutable after creation**. If you got it wrong, `helm uninstall`, `kubectl delete pvc -l ...`, then `helm install` — `kubectl edit` won't save you.
  * **Расхождение треугольника лейблов.** Три места, которые нужно держать синхронными: `StatefulSet.spec.selector.matchLabels`, `StatefulSet.spec.template.metadata.labels` и `Service.spec.selector` (у обычного И headless Service). Селектор **неизменяем после создания**. Если ошибся — `helm uninstall`, `kubectl delete pvc -l ...`, затем `helm install`; `kubectl edit` не спасёт.

- **Confusing `serviceName` with the regular Service.** Your StatefulSet's `serviceName` field points at the **headless** Service, not at the regular ClusterIP Service from Lab 9. They are different Services with different selectors satisfied by the same pods. The regular Service is for `curl http://web/` from anywhere in the cluster (load-balanced). The headless Service is for `curl http://web-0.web-headless/` to address a specific pod.
  * **Путаница `serviceName` с обычным Service.** Поле `serviceName` твоего StatefulSet указывает на **headless** Service, а не на обычный ClusterIP Service из лабы 9. Это разные Service с разными селекторами, которым соответствуют одни и те же поды. Обычный Service — для `curl http://web/` из любой точки кластера (с балансировкой). Headless Service — для `curl http://web-0.web-headless/`, чтобы обратиться к конкретному поду.

</details>

<details>
<summary>🛠️ Useful CLI (no YAML — recipes only)</summary>

🛠️ Полезный CLI (без YAML — только рецепты)

```bash
# Inspect what the cluster offers before you write YAML
kubectl get storageclass
kubectl get sc local-path -o yaml | grep -E 'provisioner|reclaimPolicy|volumeBindingMode|allowVolumeExpansion'

# The four together tell the whole StatefulSet story
kubectl get sts,pod,pvc,svc -n lab15

# Confirm per-pod PVC binding (the headline)
kubectl get pvc -n lab15 -l app.kubernetes.io/name=lab10-app
# Expect: data-<sts>-0, data-<sts>-1, data-<sts>-2 — all Bound

# Per-pod DNS without nslookup
kubectl exec -n lab15 <sts>-0 -- getent hosts <sts>-1.<headless>.lab15.svc.cluster.local
# Or with full tooling:
kubectl run net-debug --rm -it --image=nicolaka/netshoot --restart=Never -n lab15 -- bash

# Persistence proof (Task 4)
kubectl exec -n lab15 <sts>-1 -- sh -c 'echo "MARKER-$(date +%s)" > /data/marker'
kubectl exec -n lab15 <sts>-1 -- cat /data/marker
kubectl delete pod -n lab15 <sts>-1
kubectl wait --for=condition=Ready pod/<sts>-1 -n lab15 --timeout=60s
kubectl exec -n lab15 <sts>-1 -- cat /data/marker   # MUST match the value above
```

Пояснение: это шпаргалка команд для проверки StorageClass, привязки PVC, DNS для отдельных подов и главного доказательства сохранности; менять YAML здесь не нужно.

</details>

---

## Looking Ahead

Что впереди

| Lab | What it adds to this stack |
|---:|---|
| 16 | **kube-prometheus-stack** — scrape per-pod metrics from your StatefulSet via a `ServiceMonitor` |
| 17 *(bonus)* | **Cloudflare Workers** — when the cluster is overkill: V8 isolates, instant cold-start |
| 18 *(bonus)* | **Nix** — reproducible builds and dev shells without a container in sight |

| Лаба | Что добавляет в этот стек |
|---:|---|
| 16 | **kube-prometheus-stack** — сбор метрик на каждый под из твоего StatefulSet через `ServiceMonitor` |
| 17 *(бонус)* | **Cloudflare Workers** — когда кластер избыточен: изоляты V8, мгновенный холодный старт |
| 18 *(бонус)* | **Nix** — воспроизводимые сборки и dev-окружения без контейнеров |

---

**Good luck!** 💾

**Удачи!** 💾

> **Remember:** StatefulSets are *managed pets* — stable identity + per-pod storage. The whole lab boils down to one capture: write a marker to `web-1`, delete `web-1`, read the marker back from the new `web-1` — same value, because `data-web-1` was bound to ordinal 1, not to the pod that lived there. If only one capture survives the lab, make it that one.
>
> **Помни:** StatefulSets — это *управляемые питомцы*: стабильная идентичность + хранилище на каждый под. Вся лаба сводится к одной записи: запиши маркер в `web-1`, удали `web-1`, прочитай маркер из нового `web-1` — значение то же, потому что `data-web-1` был привязан к порядковому номеру 1, а не к поду, который там жил. Если после лабы удастся сохранить только одну запись, пусть это будет она.
