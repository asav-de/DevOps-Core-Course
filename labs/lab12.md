# Lab 12 — ConfigMaps & Persistent Volumes

Лаба 12 — ConfigMaps и постоянные тома (Persistent Volumes)

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Configuration%20%26%20Storage-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-ConfigMaps%20%7C%20PVC%20%7C%20K8s%201.36-informational)

> **Goal:** Externalize non-secret config with ConfigMaps and make application data survive `kubectl delete pod` with a PersistentVolumeClaim.
> **Deliverable:** A PR from `lab12` extending your Lab 10/11 chart with two ConfigMaps + a PVC, an app-side visit counter, and `docs/LAB12.md` containing the write-delete-pod-read-from-new-pod persistence proof.
>
> **Цель:** вынести несекретную конфигурацию в ConfigMaps и сделать так, чтобы данные приложения переживали `kubectl delete pod` благодаря PersistentVolumeClaim.
> **Что нужно сдать:** PR из ветки `lab12`, который расширяет твой чарт из лаб 10/11 двумя ConfigMap и PVC, счётчиком посещений в приложении и файлом `docs/LAB12.md` с доказательством сохранности данных по схеме «записать — удалить под — прочитать из нового пода».

---

## Overview

Обзор

Lab 11 hid your *secrets* (DB passwords, API tokens). But not every value is a secret — log levels, feature flags, app names, `NGINX` configs — these belong in **ConfigMaps**. And containers are *ephemeral*: when a pod restarts, its writable layer is gone. Anything your app wrote to local disk vanishes. The fix is a **PersistentVolumeClaim** — the pod claims storage by name, the volume follows the pod from one node to another, and the data outlives any individual pod.

В лабе 11 ты прятал *секреты* (пароли к БД, API-токены). Но секрет — не любое значение: уровни логирования, фиче-флаги, имена приложений, конфиги `NGINX` относятся к **ConfigMaps**. А контейнеры *эфемерны*: при перезапуске пода его записываемый слой пропадает, и всё, что приложение записало на локальный диск, исчезает. Решение — **PersistentVolumeClaim**: под запрашивает хранилище по имени, том следует за подом с одного узла на другой, а данные живут дольше любого отдельного пода.

In this lab you will practice:

В этой лабе ты отработаешь:

- The three ConfigMap injection patterns (single env, `envFrom` bulk, volume mount) and **picking the right one for the situation**
  * Три способа внедрения ConfigMap (одна переменная окружения, массово через `envFrom`, монтирование тома) и **выбор подходящего под ситуацию**
- Designing a PVC: access mode, size, storage class — by **reading what the cluster actually offers** instead of guessing
  * Проектирование PVC: режим доступа, размер, класс хранилища — на основе того, **что реально предлагает кластер**, а не догадок
- The headline proof: **write → `kubectl delete pod` → read from the new pod → same value**. A PVC that merely `Bound`s does NOT prove persistence.
  * Главное доказательство: **записать → `kubectl delete pod` → прочитать из нового пода → то же значение**. PVC, который всего лишь перешёл в `Bound`, сохранность данных НЕ доказывает.

> ⚠️ **Scope:** one replica with `ReadWriteOnce` storage. `ReadWriteMany` (multi-node concurrent writes) needs a network FS (NFS/EFS/CephFS) and is outside this lab. StatefulSets + `volumeClaimTemplates` (one PVC per replica) come in Lab 15.
>
> ⚠️ **Границы лабы:** одна реплика с хранилищем `ReadWriteOnce`. Для `ReadWriteMany` (параллельная запись с нескольких узлов) нужна сетевая ФС (NFS/EFS/CephFS), и это за рамками лабы. StatefulSets и `volumeClaimTemplates` (один PVC на реплику) появятся в лабе 15.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**У тебя должно быть с прошлых лаб:**

- Lab 9 — your `web` Deployment + Service on k3d (1.36)
  * Лаба 9 — твои Deployment `web` и Service в k3d (1.36)
- Lab 10 — your `lab10-app` Helm chart packaging that Deployment + Service
  * Лаба 10 — твой Helm-чарт `lab10-app`, упаковывающий эти Deployment и Service
- Lab 11 — Secret(s) wired into the same chart
  * Лаба 11 — Secret(ы), подключённые в тот же чарт

**This lab adds (in your existing chart):**

**Эта лаба добавляет (в твой существующий чарт):**

- App-side **visit counter** that writes to a file under `DATA_DIR` (default `/data`)
  * **Счётчик посещений** в приложении, который пишет в файл в каталоге `DATA_DIR` (по умолчанию `/data`)
- `templates/configmap.yaml` — two ConfigMaps; one mounted as a file, one bulk-injected as env vars
  * `templates/configmap.yaml` — два ConfigMap: один монтируется как файл, второй массово внедряется как переменные окружения
- `templates/pvc.yaml` — a PVC under `.Values.persistence.enabled`
  * `templates/pvc.yaml` — PVC, создаваемый при `.Values.persistence.enabled`
- Deployment patches to wire them in
  * Патчи Deployment, чтобы всё это подключить
- `docs/LAB12.md` — the persistence proof + your design reasoning
  * `docs/LAB12.md` — доказательство сохранности данных и обоснование твоих решений

> 📚 Pairs with **Lecture 12 — ConfigMaps and Persistent Volumes**. Re-read slides 6–8 (injection patterns), 13–15 (PV/PVC/StorageClass, access modes, reclaim policies), and 16 (`subPath` trap) before you start.
>
> 📚 Пара к **лекции 12 — ConfigMaps и постоянные тома**. Перед началом перечитай слайды 6–8 (способы внедрения), 13–15 (PV/PVC/StorageClass, режимы доступа, политики reclaim) и 16 (ловушка `subPath`).

---

## Setup

Подготовка

You need the k3d cluster from Lab 9 running on Kubernetes 1.36, your `lab10-app` chart, and `kubectl`.

Тебе понадобится кластер k3d из лабы 9 на Kubernetes 1.36, твой чарт `lab10-app` и `kubectl`.

```bash
kubectl get nodes                # should show k3d-devops-server-0 + 2 agents on v1.36.x
helm list -n lab12 2>/dev/null   # namespace you'll use below
```

Пояснение: первая команда показывает узлы кластера — сервер `k3d-devops-server-0` и двух агентов на v1.36.x; вторая проверяет, есть ли уже релизы Helm в неймспейсе `lab12`.

**Before you write a single line of YAML**, inspect what the cluster gives you for free — the answers to "which access mode?" and "which storage class?" depend on it:

**Прежде чем написать хотя бы строку YAML**, посмотри, что кластер даёт тебе бесплатно: ответы на вопросы «какой режим доступа?» и «какой класс хранилища?» зависят именно от этого:

```bash
kubectl get storageclass
# YOUR TASK: read the output. Which one has the (default) marker? What provisioner backs it?
# Write the answer in docs/LAB12.md — this is the storage class your PVC will use.
```

Пояснение: `kubectl get storageclass` выводит доступные классы хранилища; в комментариях к коду — задание: найди класс с пометкой `(default)`, определи его provisioner и запиши вывод в `docs/LAB12.md` — именно этот класс будет использовать твой PVC.

> 💡 k3d (k3s) ships **`local-path`** (`rancher.io/local-path`) marked `(default)`. It's RWO-only, no snapshots — fine for a single-pod visits counter, not what you'd run in prod. On a real cloud cluster the same chart would bind to an **AWS EBS** (`ebs.csi.aws.com`) or **GCE PD** (`pd.csi.storage.gke.io`) volume via its CSI driver, with no chart change.
>
> 💡 В k3d (k3s) по умолчанию поставляется **`local-path`** (`rancher.io/local-path`) с пометкой `(default)`. Он поддерживает только RWO и не умеет снапшоты — для счётчика посещений в одном поде это нормально, но в проде так не делают. В настоящем облачном кластере тот же чарт через CSI-драйвер привяжется к тому **AWS EBS** (`ebs.csi.aws.com`) или **GCE PD** (`pd.csi.storage.gke.io`) без единой правки чарта.

---

## Task 1 — Application Persistence Upgrade (2 pts)

Задача 1 — обновление приложения для работы с постоянным хранилищем (2 балла)

Before any Kubernetes work, the app needs something worth persisting. Add a visit counter that writes to a file, and prove it works locally before you move on.

Прежде чем браться за Kubernetes, приложению нужно что-то, что стоит сохранять. Добавь счётчик посещений, который пишет в файл, и убедись, что он работает локально, прежде чем идти дальше.

### 1.1 — Add the visit counter

1.1 — Добавляем счётчик посещений

`YOUR TASK`: in `app_python/app.py` (or your language equivalent), add the counter.

`YOUR TASK`: в `app_python/app.py` (или эквивалентном файле на твоём языке) добавь счётчик.

Requirements:

Требования:

- `GET /` increments a counter; the value is stored in a file under `DATA_DIR` (env var, default `/data`).
  * `GET /` увеличивает счётчик; значение хранится в файле в каталоге `DATA_DIR` (переменная окружения, по умолчанию `/data`).
- On startup, read the file. If it's missing, default to `0` — don't crash.
  * При старте читай файл. Если его нет — начинай с `0`, не падай.
- Add `GET /visits` returning `{"visits": N}` JSON.
  * Добавь `GET /visits`, возвращающий JSON `{"visits": N}`.
- Write **atomically**: write to a temp file in the same directory, then `os.replace(tmp, final)`. A crash mid-write must not corrupt the count.
  * Пиши **атомарно**: сначала во временный файл в том же каталоге, затем `os.replace(tmp, final)`. Падение посреди записи не должно портить счётчик.
- Make the data dir on startup if it doesn't exist (`os.makedirs(..., exist_ok=True)`).
  * Создавай каталог данных при старте, если его нет (`os.makedirs(..., exist_ok=True)`).

Hints — sketch of the moving parts (do NOT copy-paste, derive the code yourself):

Подсказки — набросок составных частей (НЕ копируй, напиши код сам):

- **constants** — `DATA_DIR = env("DATA_DIR","/data")`, `COUNTER = DATA_DIR + "/visits"`
  * **константы** — `DATA_DIR = env("DATA_DIR","/data")`, `COUNTER = DATA_DIR + "/visits"`
- **read_count** — open + `int(...)`; on `FileNotFoundError` return `0`
  * **read_count** — открыть файл и `int(...)`; при `FileNotFoundError` вернуть `0`
- **write_count(n)** — `mkstemp` in `DATA_DIR` → write → `os.replace(tmp, COUNTER)` (atomic)
  * **write_count(n)** — `mkstemp` в `DATA_DIR` → запись → `os.replace(tmp, COUNTER)` (атомарно)
- **`GET /`** — `c = read_count() + 1`; `write_count(c)`; respond
  * **`GET /`** — `c = read_count() + 1`; `write_count(c)`; ответить
- **`GET /visits`** — respond `{ "visits": read_count() }`
  * **`GET /visits`** — ответить `{ "visits": read_count() }`

### 1.2 — Local Docker Compose persistence test

1.2 — Локальный тест сохранности данных в Docker Compose

`YOUR TASK`: in `app_python/docker-compose.yml`, mount a host directory at `DATA_DIR`. Hit `/` three times, restart the container, hit `/visits` — the count must continue, not reset.

`YOUR TASK`: в `app_python/docker-compose.yml` смонтируй каталог хоста в `DATA_DIR`. Трижды запроси `/`, перезапусти контейнер, запроси `/visits` — счётчик должен продолжиться, а не сброситься.

Skeleton (fill in the blanks):

Заготовка (заполни пропуски):

```yaml
services:
  app:
    # ... your build/image from Lab 2 ...
    environment:
      DATA_DIR: # YOUR TASK
    volumes:
      - # YOUR TASK: host-path : DATA_DIR
```

Пояснение: в `environment` укажи `DATA_DIR`, а в `volumes` привяжи каталог хоста к этому же пути — тогда данные останутся на диске хоста при перезапуске контейнера.

### 1.3 — Proof of work

1.3 — Подтверждение работы

Paste into `docs/LAB12.md`:

Вставь в `docs/LAB12.md`:

- The three curls (`curl localhost:8080/` × 3) + the `/visits` reading.
  * Три запроса `curl` (`curl localhost:8080/` × 3) и показание `/visits`.
- `docker compose restart app` followed by `/visits` showing the **same** count, not 0.
  * `docker compose restart app`, после чего `/visits` показывает **то же** число, а не 0.
- `cat ./data/visits` from the host showing the same integer (proves it's actually persisted, not a race).
  * `cat ./data/visits` с хоста показывает то же целое число (доказывает, что данные действительно сохранены, а не совпали случайно).

---

## Task 2 — ConfigMaps: Three Patterns, One Decision Per Value (3 pts)

Задача 2 — ConfigMaps: три способа внедрения и одно решение для каждого значения (3 балла)

The lecture covered three ways to inject a ConfigMap into a pod. **One pattern per *value* — not all three for every value.** You'll ship two ConfigMaps (one shaped as files, one shaped as env vars), wire each into the pod with the pattern that fits its shape, and defend the choice for the three example values in the decision table below.

На лекции разобраны три способа внедрить ConfigMap в под. **Один способ на *значение* — не все три для каждого значения.** Ты выпустишь два ConfigMap (один в виде файлов, второй в виде переменных окружения), подключишь каждый тем способом, который подходит под его форму, и обоснуешь выбор для трёх примеров из таблицы решений ниже.

### 2.1 — Author the two ConfigMaps

2.1 — Создаём два ConfigMap

You'll ship two ConfigMaps in `templates/configmap.yaml`. The shape and *labels* are given; the **data** is yours to fill from the chart's `values.yaml`.

Ты создашь два ConfigMap в `templates/configmap.yaml`. Структура и *метки* даны; **данные** ты заполняешь сам из `values.yaml` чарта.

Skeleton — `templates/configmap.yaml` (one file, two CMs separated by `---`):

Заготовка — `templates/configmap.yaml` (один файл, два ConfigMap, разделённые `---`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "lab10-app.fullname" . }}-file
  labels: {{- include "lab10-app.labels" . | nindent 4 }}
data:
  config.json: |-                # YOUR TASK: load files/config.json via .Files.Get
                                 #            and indent 4 — hint: `| indent 4`
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "lab10-app.fullname" . }}-env
  labels: {{- include "lab10-app.labels" . | nindent 4 }}
data:                            # YOUR TASK: ≥3 K/V pairs from .Values
                                 # ⚠️ keys must match [A-Z_][A-Z0-9_]* (Lec 12 slide 7)
                                 # invalid names are silently skipped by envFrom
```

Пояснение: в первом ConfigMap файл `config.json` подгружается через `.Files.Get` и сдвигается на 4 пробела; во втором — не менее трёх пар ключ-значение из `.Values`, причём имена ключей должны соответствовать шаблону `[A-Z_][A-Z0-9_]*`, иначе `envFrom` молча их пропустит.

`YOUR TASK`: create `files/config.json` with at least these keys: `appName`, `environment`, `features` (object with `visitsCounter: true`). Add the corresponding values to `values.yaml`.

`YOUR TASK`: создай `files/config.json` как минимум с ключами `appName`, `environment`, `features` (объект с `visitsCounter: true`). Добавь соответствующие значения в `values.yaml`.

### 2.2 — Pick an injection pattern per value

2.2 — Выбираем способ внедрения для каждого значения

Three injection patterns exist (Lecture 12, slides 6–8). Below is the skeleton of each as a separate Deployment-patch block — **keep the ones you need, delete the others**, then justify your choices in `docs/LAB12.md`. At minimum your Deployment ends up using Pattern B (the env-shaped CM) **and** Pattern C (the file-shaped CM); Pattern A is optional and only earns points if you can defend a specific reason to rename or single-pick a key.

Есть три способа внедрения (лекция 12, слайды 6–8). Ниже заготовка каждого в виде отдельного фрагмента патча Deployment: **оставь нужные, удали остальные**, а затем обоснуй выбор в `docs/LAB12.md`. Как минимум твой Deployment должен использовать способ B (ConfigMap в виде переменных окружения) **и** способ C (ConfigMap в виде файлов); способ A необязателен и даёт баллы, только если ты сможешь обосновать, зачем переименовывать или выбирать отдельный ключ.

**Pattern A — single env, renamed/optional control.** Use when you want ONE value, possibly RENAMED into the container.

**Способ A — одна переменная окружения, с переименованием или необязательностью.** Нужен, когда тебе требуется ОДНО значение, возможно ПЕРЕИМЕНОВАННОЕ внутри контейнера.

```yaml
env:
  - name:                            # YOUR TASK: env name inside the container
    valueFrom:
      configMapKeyRef:
        name:                        # YOUR TASK: which of your two ConfigMaps holds this?
        key:                         # YOUR TASK: key in the ConfigMap (may differ from name)
        optional:                    # YOUR TASK: true/false — start if missing?
```

Пояснение: заполни имя переменной в контейнере, имя ConfigMap, ключ в нём и флаг `optional`; значение попадёт в контейнер под нужным тебе именем.

**Pattern B — `envFrom` (bulk).** EVERY key in the CM becomes an env var. No rename, no filter — curate the ConfigMap, not the injection.

**Способ B — `envFrom` (массово).** КАЖДЫЙ ключ ConfigMap становится переменной окружения. Без переименований и фильтров — наводи порядок в самом ConfigMap, а не во внедрении.

```yaml
envFrom:
  - configMapRef:
      name:                          # YOUR TASK
```

Пояснение: достаточно указать имя ConfigMap — все его ключи с допустимыми именами станут переменными окружения пода.

**Pattern C — volume mount (whole directory, NOT `subPath`).** For structured files (JSON/YAML/conf) the app reads; auto-updates without an image rebuild (Bonus depends on this).

**Способ C — монтирование тома (весь каталог, БЕЗ `subPath`).** Для структурированных файлов (JSON/YAML/conf), которые читает приложение; обновляются автоматически без пересборки образа (от этого зависит бонусная часть).

```yaml
volumeMounts:
  - name:                            # YOUR TASK: must match the volume name below
    mountPath:                       # YOUR TASK: dir where config.json appears
                                     # ⚠️ NO subPath: here — see Bonus + Pitfalls
volumes:
  - name:                            # YOUR TASK: same name as the volumeMount above
    configMap:
      name:                          # YOUR TASK: which ConfigMap supplies the files?
```

Пояснение: имя тома должно совпадать в `volumeMounts` и `volumes`, `mountPath` — каталог, где появится `config.json`; `subPath` здесь быть не должно.

Pick the right pattern for each of these values and document the choice in `docs/LAB12.md` (one sentence each):

Выбери подходящий способ для каждого из значений и опиши выбор в `docs/LAB12.md` (по одному предложению на каждое):

| Value | Pattern? | Why? |
|---|---|---|
| `LOG_LEVEL` (a single string the framework reads from env) | YOUR TASK | YOUR TASK |
| `config.json` (a structured file with feature flags) | YOUR TASK | YOUR TASK |
| ≥ 10 boolean feature flags that all need to be env vars | YOUR TASK | YOUR TASK |

| Значение | Способ? | Почему? |
|---|---|---|
| `LOG_LEVEL` (одна строка, которую фреймворк читает из окружения) | YOUR TASK | YOUR TASK |
| `config.json` (структурированный файл с фиче-флагами) | YOUR TASK | YOUR TASK |
| ≥ 10 булевых фиче-флагов, которые все должны быть переменными окружения | YOUR TASK | YOUR TASK |

> 💡 **Hint** — none of the three answers is "all three". One value, one pattern. The point is the *decision*, not the proliferation.
>
> 💡 **Подсказка:** ни один из трёх ответов не «все три». Одно значение — один способ. Смысл в *решении*, а не в количестве.

### 2.3 — Proof of work

2.3 — Подтверждение работы

`YOUR TASK`: after `helm upgrade --install lab10-app k8s/lab10-app -n lab12 --create-namespace` (same chart path as Lab 10/11), capture:

`YOUR TASK`: после `helm upgrade --install lab10-app k8s/lab10-app -n lab12 --create-namespace` (тот же путь к чарту, что в лабах 10/11) сними:

```bash
# YOUR TASK: get the pod name into $POD
# YOUR TASK: cat the mounted config.json from inside the pod — shows file mount works
# YOUR TASK: printenv | grep -E '<your env keys>' — shows env injection works
# YOUR TASK: kubectl describe pod $POD | grep -A2 -E 'Environment|Mounts' — shows the wiring
```

Пояснение: это четыре команды, которые нужно восстановить самому — получить имя пода, прочитать смонтированный `config.json` внутри пода, вывести переменные окружения и показать связку через `kubectl describe pod`.

Paste into `docs/LAB12.md`:

Вставь в `docs/LAB12.md`:

- The three captures above (file contents, env vars, pod description excerpt)
  * Три снимка выше (содержимое файла, переменные окружения, фрагмент описания пода)
- The decision table from 2.2 filled in
  * Заполненная таблица решений из 2.2
- One paragraph: *if the ConfigMap had 30 keys but you only needed 2, would you still use `envFrom`? Why/why not?*
  * Один абзац: *если бы в ConfigMap было 30 ключей, а нужны тебе только 2, использовал бы ты всё равно `envFrom`? Почему да или нет?*

---

## Task 3 — Persistent Volumes: Read the Cluster, Then Write the Claim (3 pts)

Задача 3 — Persistent Volumes: сначала читаем кластер, потом пишем заявку на том (3 балла)

A PVC has four knobs: `accessModes`, `resources.requests.storage`, `storageClassName`, and (less obviously) the **reclaim policy** inherited from the StorageClass. You'll set the first three explicitly — but only after inspecting what the cluster already offers.

У PVC четыре параметра: `accessModes`, `resources.requests.storage`, `storageClassName` и (менее очевидно) **политика reclaim**, унаследованная от StorageClass. Первые три ты задашь явно — но только после того, как посмотришь, что уже предлагает кластер.

### 3.1 — Inspect the cluster's storage first

3.1 — Сначала изучаем хранилище кластера

```bash
kubectl get storageclass
kubectl get sc <default-sc-name> -o yaml | grep -E 'provisioner|reclaimPolicy|volumeBindingMode'
```

Пояснение: первая команда перечисляет классы хранилища, вторая показывает у выбранного класса provisioner, политику reclaim и режим привязки тома.

`YOUR TASK`: in `docs/LAB12.md`, answer:

`YOUR TASK`: ответь в `docs/LAB12.md`:

- Which StorageClass has the `(default)` marker? What provisioner backs it?
  * У какого StorageClass стоит пометка `(default)`? Какой provisioner за ним стоит?
- What's its `reclaimPolicy` (Delete vs Retain)? What happens to your data if you delete the PVC?
  * Какая у него `reclaimPolicy` (`Delete` или `Retain`)? Что случится с твоими данными, если удалить PVC?
- What's its `volumeBindingMode`? Why does `WaitForFirstConsumer` mean a `Pending` PVC is **normal**, not a bug?
  * Каков его `volumeBindingMode`? Почему при `WaitForFirstConsumer` статус PVC `Pending` — это **нормально**, а не баг?

### 3.2 — Pick `accessModes` for the visits-counter use case

3.2 — Выбираем `accessModes` для сценария со счётчиком посещений

The four modes (Lecture 12, slide 14):

Четыре режима (лекция 12, слайд 14):

- **`ReadWriteOnce`** — RW from one *node* (multiple pods on that node OK). Cloud block storage (EBS/PD/Azure Disk) only supports this.
  * **`ReadWriteOnce`** — чтение и запись с одного *узла* (несколько подов на этом узле допустимы). Облачное блочное хранилище (EBS/PD/Azure Disk) поддерживает только его.
- **`ReadOnlyMany`** — RO from many nodes. Content distribution.
  * **`ReadOnlyMany`** — только чтение со многих узлов. Для раздачи контента.
- **`ReadWriteMany`** — RW from many nodes simultaneously. Needs NFS/EFS/CephFS — slower, more expensive.
  * **`ReadWriteMany`** — чтение и запись одновременно со многих узлов. Нужна NFS/EFS/CephFS — медленнее и дороже.
- **`ReadWriteOncePod`** — RW from exactly *one* pod cluster-wide (GA in K8s 1.29).
  * **`ReadWriteOncePod`** — чтение и запись ровно из *одного* пода во всём кластере (GA в K8s 1.29).

`YOUR TASK`: pick ONE for the visits counter and justify it in `docs/LAB12.md`. Hint: you have one Deployment replica writing to one file. Don't reach for the most flexible option — reach for the one that actually fits.

`YOUR TASK`: выбери ОДИН режим для счётчика посещений и обоснуй его в `docs/LAB12.md`. Подсказка: у тебя одна реплика Deployment, которая пишет в один файл. Не бери самый гибкий вариант — бери тот, что действительно подходит.

### 3.3 — Author the PVC

3.3 — Пишем манифест PVC

Skeleton — `templates/pvc.yaml`:

Заготовка — `templates/pvc.yaml`:

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "lab10-app.fullname" . }}-data
  labels: {{- include "lab10-app.labels" . | nindent 4 }}
spec:
  accessModes:
    -                              # YOUR TASK: RWO | ROX | RWX | RWOP — pick + defend
  resources:
    requests:
      storage:                     # YOUR TASK: size — counter is one int, don't ask 100Gi
  {{- with .Values.persistence.storageClass }}
  storageClassName: {{ . }}        # omit field → cluster default; "" → also default
  {{- end }}
{{- end }}
```

Пояснение: PVC создаётся только при включённой настройке `persistence.enabled`; заполни `accessModes` и размер запроса, а `storageClassName` оставь пустым, чтобы взять класс по умолчанию, или укажи конкретный.

Add to `values.yaml`:

Добавь в `values.yaml`:

```yaml
persistence:
  enabled: true
  size: # YOUR TASK
  storageClass: # YOUR TASK: "" to use the cluster default, or a specific name (e.g. local-path)
```

Пояснение: значения `enabled`, `size` и `storageClass` управляют PVC из шаблона; пустая строка в `storageClass` означает класс по умолчанию.

### 3.4 — Mount it on the Deployment

3.4 — Монтируем том в Deployment

Patch `templates/deployment.yaml` — only the new bits:

Измени `templates/deployment.yaml` — только новые фрагменты:

```yaml
volumeMounts:
  - name:                          # YOUR TASK: invent a volume name (match below)
    mountPath:                     # YOUR TASK: must equal DATA_DIR from Task 1
volumes:
  - name:                          # YOUR TASK: same name as the volumeMount above
    {{- if .Values.persistence.enabled }}
    persistentVolumeClaim:
      claimName:                   # YOUR TASK: same name as the PVC you authored above
    {{- else }}
    emptyDir: {}                   # fallback: data dies with the pod
    {{- end }}
```

Пояснение: `volumeMounts` и `volumes` должны ссылаться на одно имя; `mountPath` обязан совпадать с `DATA_DIR` из задачи 1, а `claimName` — с именем PVC. Если persistence выключена, используется `emptyDir`, и данные исчезают вместе с подом.

### 3.5 — The persistence proof (the headline)

3.5 — Главное доказательство сохранности данных

This is the heart of the lab. **A `Bound` PVC means a volume exists, NOT that your data survived a pod restart.** You prove persistence by writing, deleting the pod, and reading from the new pod.

Это сердце лабы. **Статус PVC `Bound` означает лишь, что том существует, а НЕ то, что данные пережили перезапуск пода.** Сохранность доказывается так: записать, удалить под и прочитать из нового пода.

`YOUR TASK`: figure out the four steps yourself. You'll need `kubectl exec ... -- sh -c 'echo ... > $DATA_DIR/marker'`, `kubectl delete pod`, `kubectl wait`, and a final `kubectl exec ... cat $DATA_DIR/marker`.

`YOUR TASK`: продумай четыре шага сам. Тебе понадобятся `kubectl exec ... -- sh -c 'echo ... > $DATA_DIR/marker'`, `kubectl delete pod`, `kubectl wait` и финальный `kubectl exec ... cat $DATA_DIR/marker`.

Use a **separate file** called `marker` for this proof — don't reuse `/data/visits`. The counter increments naturally and you want a constant string to compare. Both files live on the same PVC, so the proof for `marker` is also the proof for `visits`.

Для доказательства используй **отдельный файл** `marker`, не переиспользуй `/data/visits`: счётчик увеличивается естественным образом, а тебе нужна неизменная строка для сравнения. Оба файла лежат на одном PVC, поэтому доказательство для `marker` — это и доказательство для `visits`.

```bash
# YOUR TASK: write a unique marker (e.g. epoch seconds) to $DATA_DIR/marker
#            inside the running pod (DATA_DIR defaults to /data — adjust if you changed it)
# YOUR TASK: delete the pod and wait for the Deployment to spin up a fresh one
# YOUR TASK: read $DATA_DIR/marker from the NEW pod
# YOUR TASK: prove the marker is the SAME — the write survived the pod's death
```

Пояснение: четыре шага — записать уникальный маркер (например, секунды эпохи) в `$DATA_DIR/marker`, удалить под и дождаться, пока Deployment создаст новый, прочитать маркер из нового пода и убедиться, что значение совпадает.

The deliverable is the actual captured output showing the *same* string before and after. If it differs (or the second `cat` returns "No such file or directory"), your PVC isn't really mounted at the data dir, or your storage class is doing something unexpected — debug with `kubectl describe pvc` and `kubectl describe pod`.

Сдать нужно реальный вывод, где до и после видна *одна и та же* строка. Если она отличается (или второй `cat` возвращает «No such file or directory»), значит PVC на самом деле не смонтирован в каталог данных либо класс хранилища ведёт себя неожиданно — отлаживай через `kubectl describe pvc` и `kubectl describe pod`.

> 💡 You may see `Unknown stream id 5` from `kubectl exec` — that's a harmless client-side websocket warning, not a failure. If the data round-trips, you're fine.
>
> 💡 Ты можешь увидеть `Unknown stream id 5` от `kubectl exec` — это безобидное клиентское предупреждение websocket, а не сбой. Если данные считываются обратно, всё в порядке.

### 3.6 — Proof of work

3.6 — Подтверждение работы

Paste into `docs/LAB12.md`:

Вставь в `docs/LAB12.md`:

- The storage-class inspection from 3.1 + your answers
  * Вывод проверки StorageClass из 3.1 и твои ответы
- The access-mode decision from 3.2 with justification
  * Выбранный в 3.2 режим доступа с обоснованием
- `kubectl get pvc` showing the claim `Bound` (or `Pending` if `WaitForFirstConsumer` and no pod has consumed it yet — explain which)
  * Вывод `kubectl get pvc` со статусом `Bound` (или `Pending`, если действует `WaitForFirstConsumer` и под ещё не использовал заявку, — объясни, какой случай у тебя)
- **The persistence proof from 3.5: original pod name, marker written, delete command, new pod name, marker read — same value, side by side**
  * **Доказательство сохранности из 3.5: имя исходного пода, записанный маркер, команда удаления, имя нового пода, прочитанный маркер — одинаковое значение рядом**

---

## Task 4 — Documentation (2 pts)

Задача 4 — Документация (2 балла)

Bundle everything into `docs/LAB12.md`. Required sections, in order:

Собери всё в `docs/LAB12.md`. Обязательные разделы по порядку:

1. **App changes** — counter design, `DATA_DIR`, the atomic-write rationale, the Compose persistence capture from Task 1
   * **Изменения в приложении** — устройство счётчика, `DATA_DIR`, обоснование атомарной записи, снимок сохранности в Compose из задачи 1
2. **ConfigMap design** — your `files/config.json`, the two ConfigMaps, the decision table from Task 2.2, the verification captures from 2.3
   * **Проектирование ConfigMap** — твой `files/config.json`, два ConfigMap, таблица решений из задачи 2.2, снимки проверки из 2.3
3. **PVC design** — the StorageClass inspection, the access-mode justification, the PVC YAML, the mount, **the headline persistence proof from 3.5**
   * **Проектирование PVC** — проверка StorageClass, обоснование режима доступа, YAML PVC, монтирование, **главное доказательство сохранности из 3.5**
4. **ConfigMap vs Secret** — when to use each (RBAC scope, tmpfs vs regular fs, encryption-at-rest story). Reference Lab 11.
   * **ConfigMap против Secret** — когда что использовать (область RBAC, tmpfs против обычной ФС, шифрование при хранении). Сошлись на лабу 11.
5. **Common pitfalls you hit** — at least one real one with how you debugged it
   * **Грабли, на которые ты наступил** — хотя бы одни реальные, с рассказом, как ты их отлаживал

---

## Bonus Task — ConfigMap Hot Reload (2 pts)

Бонусная задача — горячая перезагрузка ConfigMap (2 балла)

Edit a value in `lab10-app-env`. Watch the running pod. The env var **doesn't change**. Edit a value in `lab10-app-file`. Watch the mounted file. It **eventually** changes — minutes later. If you'd used `subPath` to mount it as a single file? It would **never** change.

Измени значение в `lab10-app-env` и понаблюдай за работающим подом: переменная окружения **не изменится**. Измени значение в `lab10-app-file` и следи за смонтированным файлом: он **в конце концов** изменится — через несколько минут. А если смонтировать файл через `subPath`? Он **не изменится никогда**.

These three behaviors are why every team eventually picks a hot-reload strategy. Your job is to:

Эти три модели поведения и заставляют каждую команду рано или поздно выбрать стратегию горячей перезагрузки. Твоя задача:

1. **Reproduce the default behavior.** `YOUR TASK`: edit a value with `kubectl edit configmap`, time how long until (a) the env var inside a running pod reflects the change — measure in the pod with `printenv` — and (b) the mounted file reflects the change — measure with `cat` in a loop. Document the result.
   * **Воспроизведи поведение по умолчанию.** `YOUR TASK`: измени значение через `kubectl edit configmap` и замерь, через сколько времени: (а) переменная окружения внутри работающего пода отразит изменение — проверяй в поде через `printenv`; (б) смонтированный файл отразит изменение — замеряй через `cat` в цикле. Задокументируй результат.

2. **Explain `subPath`.** `YOUR TASK`: in one paragraph in `docs/LAB12.md`, explain why a `subPath` mount is a one-shot copy that never updates, and when you'd accept that trade-off anyway (hint: clean mount paths, no `..data` symlink visible to the app).
   * **Объясни `subPath`.** `YOUR TASK`: одним абзацем в `docs/LAB12.md` объясни, почему монтирование через `subPath` — это разовая копия, которая никогда не обновляется, и когда ты бы всё же согласился на такой компромисс (подсказка: чистые пути монтирования, приложению не видна симлинк `..data`).

3. **Pick and implement ONE strategy.** Don't reach for the install command yet — research the three options and choose:
   * **Выбери и реализуй ОДНУ стратегию.** Не хватайся сразу за команду установки — изучи три варианта и выбери:

   - **A — Checksum annotation (GitOps-native, no extra controller):** the pod template carries an annotation whose value is the `sha256sum` of the rendered ConfigMap. Any data change → annotation change → Deployment rolls out. Implemented entirely in your Helm template.
     * **A — аннотация с хешем (родная для GitOps, без лишнего контроллера):** в шаблоне пода есть аннотация, значение которой — `sha256sum` отрендеренного ConfigMap. Любое изменение данных → меняется аннотация → Deployment выполняет rollout. Реализуется целиком в твоём Helm-шаблоне.
   - **B — Reloader controller** (Stakater): an in-cluster controller watches ConfigMaps/Secrets and patches owning Deployments to trigger a rollout. Zero chart logic, one annotation.
     * **B — контроллер Reloader** (Stakater): контроллер в кластере следит за ConfigMaps/Secrets и патчит владеющие Deployment, запуская rollout. Ноль логики в чарте, одна аннотация.
   - **C — App-level watch:** the app `inotify`-watches the mounted file and re-reads on change. No restart, instant. Used by NGINX, Envoy, Prometheus, Loki natively.
     * **C — наблюдение на стороне приложения:** приложение через `inotify` следит за смонтированным файлом и перечитывает его при изменении. Без перезапуска и мгновенно. Так работают NGINX, Envoy, Prometheus и Loki из коробки.

   `YOUR TASK`: in `docs/LAB12.md`, write 3–4 sentences on which one you picked and **why** (think: GitOps fit? extra moving parts? does your app even support C?). Then implement it. **No install commands or annotation snippets are shown in this lab** — go to the lecture, the docs, or the upstream README and figure it out.

   `YOUR TASK`: в `docs/LAB12.md` напиши 3–4 предложения о том, какой вариант ты выбрал и **почему** (подумай: подходит ли для GitOps? сколько добавляется движущихся частей? поддерживает ли твоё приложение вариант C?). Затем реализуй его. **В этой лабе не показаны ни команды установки, ни фрагменты аннотаций** — иди в лекцию, документацию или README проекта и разберись сам.

4. **Demonstrate it works.** `YOUR TASK`: change a ConfigMap value, observe either (a) a rolling restart for A/B or (b) an in-place re-read for C. Capture the evidence.
   * **Продемонстрируй, что это работает.** `YOUR TASK`: измени значение в ConfigMap и пронаблюдай либо (а) rolling restart для A/B, либо (б) перечитывание на месте для C. Сними доказательства.

> 💡 If you picked A, the hash must change when the ConfigMap **data** changes. A `helm upgrade` with no changes should NOT roll the pod. If both `helm upgrade --dry-run` runs produce the same annotation, you've done it right.
>
> 💡 Если ты выбрал A, хеш должен меняться при изменении **данных** ConfigMap. `helm upgrade` без изменений НЕ должен перезапускать под. Если два прогона `helm upgrade --dry-run` дают одинаковую аннотацию, ты всё сделал правильно.

---

## How to Submit

Как сдать работу

```bash
git switch -c lab12
git add app_python/ k8s/lab10-app/ docs/LAB12.md
git commit -m "feat(lab12): configmaps + persistence with hot-reload strategy"
git push -u origin lab12
```

Пояснение: создай ветку `lab12`, добавь файлы приложения, чарта и документации, закоммить и отправь ветку в свой форк.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab12` → `course-repo:master` *(reviewed)*
  * `your-fork:lab12` → `course-repo:master` *(на ревью)*
- `your-fork:lab12` → `your-fork:master` *(merges into your own main)*
  * `your-fork:lab12` → `your-fork:master` *(вливается в твой собственный main)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — counter + /visits + DATA_DIR + atomic write + Compose persistence
- [ ] Task 2 done — two ConfigMaps, three patterns understood, decision table filled in
- [ ] Task 3 done — PVC bound, mounted, write→delete-pod→read proof captured
- [ ] Task 4 done — docs/LAB12.md complete with all five sections
- [ ] Bonus done — default behavior measured, strategy chosen + implemented + demonstrated
```

Пояснение: чек-лист сдачи — задача 1 (счётчик, `/visits`, `DATA_DIR`, атомарная запись и сохранность в Compose), задача 2 (два ConfigMap, понимание трёх способов, заполненная таблица решений), задача 3 (PVC в статусе `Bound`, монтирование, доказательство «запись → удаление пода → чтение»), задача 4 (`docs/LAB12.md` со всеми пятью разделами) и бонус (замер поведения по умолчанию, выбранная, реализованная и продемонстрированная стратегия).

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (2 pts)

Задача 1 (2 балла)

- ✅ `GET /` increments + persists; `GET /visits` returns current count
  * `GET /` увеличивает счётчик и сохраняет его; `GET /visits` возвращает текущее значение
- ✅ `DATA_DIR` env var honored; atomic write used (temp + rename)
  * Переменная окружения `DATA_DIR` учитывается; запись атомарная (временный файл + переименование)
- ✅ Compose restart preserves the count (captured)
  * Перезапуск в Compose сохраняет счётчик (есть снимок)

### Task 2 (3 pts)

Задача 2 (3 балла)

- ✅ `files/config.json` + `lab10-app-file` ConfigMap (volume mount)
  * `files/config.json` и ConfigMap `lab10-app-file` (монтирование тома)
- ✅ `lab10-app-env` ConfigMap with ≥ 3 valid env-shaped keys (`envFrom`)
  * ConfigMap `lab10-app-env` с ≥ 3 корректными ключами для переменных окружения (`envFrom`)
- ✅ Decision table filled in for each of the three example values, with reasoning
  * Таблица решений заполнена для всех трёх примеров, с обоснованием
- ✅ Verification captures: file readable in pod, env vars present, `describe pod` shows the wiring
  * Снимки проверки: файл читается в поде, переменные окружения на месте, `describe pod` показывает связку

### Task 3 (3 pts)

Задача 3 (3 балла)

- ✅ StorageClass inspection documented (which one, what provisioner, what `volumeBindingMode`)
  * Проверка StorageClass задокументирована (какой класс, какой provisioner, какой `volumeBindingMode`)
- ✅ Access mode picked + justified (`ReadWriteOnce` is correct here — defend it)
  * Режим доступа выбран и обоснован (здесь правильный — `ReadWriteOnce`, но обоснуй это)
- ✅ PVC reaches `Bound` (or `Pending` explained via `WaitForFirstConsumer`)
  * PVC достигает `Bound` (или объяснено, почему `Pending` из-за `WaitForFirstConsumer`)
- ✅ **Write → delete pod → read from new pod, same marker, captured side by side**
  * **Запись → удаление пода → чтение из нового пода, тот же маркер, снимок рядом**

### Task 4 (2 pts)

Задача 4 (2 балла)

- ✅ All five sections in `docs/LAB12.md` complete with real captures (not illustrative)
  * Все пять разделов в `docs/LAB12.md` заполнены реальными снимками, а не иллюстративными

### Bonus (2 pts)

Бонус (2 балла)

- ✅ Default behavior measured: env var update delay + file update delay
  * Поведение по умолчанию замерено: задержка обновления переменной окружения и задержка обновления файла
- ✅ `subPath` trap explained in own words
  * Ловушка `subPath` объяснена своими словами
- ✅ One strategy (A/B/C) chosen with written justification, implemented, demonstrated
  * Одна стратегия (A/B/C) выбрана с письменным обоснованием, реализована и продемонстрирована

---

## Rubric

Критерии оценивания

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — App persistence | **2** | Counter + atomic write + Compose proof |
| **Task 2** — ConfigMaps | **3** | Two CMs, three-pattern understanding, decision table |
| **Task 3** — PVC | **3** | Right access mode, right size/SC, headline persistence proof |
| **Task 4** — Documentation | **2** | All five LAB12.md sections with real evidence |
| **Bonus** — Hot reload | **2** | Default behavior measured, strategy chosen + implemented + demonstrated |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — сохранность данных приложения | **2** | Счётчик + атомарная запись + доказательство в Compose |
| **Задача 2** — ConfigMaps | **3** | Два ConfigMap, понимание трёх способов, таблица решений |
| **Задача 3** — PVC | **3** | Правильный режим доступа, правильные размер и StorageClass, главное доказательство сохранности |
| **Задача 4** — документация | **2** | Все пять разделов LAB12.md с реальными доказательствами |
| **Бонус** — горячая перезагрузка | **2** | Поведение по умолчанию замерено, стратегия выбрана, реализована и продемонстрирована |
| **Всего** | **12** | 10 основных + 2 бонусных |

**Grading guide:**

**Как выставляются оценки:**
- **10/10:** All four tasks; persistence proof is unambiguous (same marker in both pods); ConfigMap decisions are defended, not just chosen
  * **10/10:** все четыре задачи; доказательство сохранности однозначно (одинаковый маркер в обоих подах); решения по ConfigMap обоснованы, а не просто приняты
- **8–9/10:** ConfigMaps + PVC work; persistence proof present but reasoning is thin or one pattern decision is wrong
  * **8–9/10:** ConfigMaps и PVC работают; доказательство сохранности есть, но обоснование слабое или одно решение о способе неверно
- **6–7/10:** PVC `Bound` but no write→delete→read proof, OR only one ConfigMap pattern used
  * **6–7/10:** PVC в статусе `Bound`, но нет доказательства «запись → удаление → чтение», ИЛИ использован только один способ для ConfigMap
- **< 6:** No persistence proof, or ConfigMap not wired into the pod
  * **< 6:** нет доказательства сохранности или ConfigMap не подключён к поду

---

## Resources

Ресурсы

<details>
<summary>📚 Documentation</summary>

📚 Документация

- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) — concepts
  * [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) — основные понятия
- [Configure a Pod with a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) — the three injection patterns
  * [Настройка пода с ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) — три способа внедрения
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — PV/PVC/StorageClass
  * [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — постоянные тома, заявки PVC и классы StorageClass
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) — `reclaimPolicy`, `volumeBindingMode`
  * [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) — классы хранилища: `reclaimPolicy`, `volumeBindingMode`
- [Access Modes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) — RWO/ROX/RWX/RWOP
  * [Режимы доступа](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) — RWO/ROX/RWX/RWOP
- [Helm `.Files.Get`](https://helm.sh/docs/chart_template_guide/accessing_files/) — embed a file into a ConfigMap
  * [Helm `.Files.Get`](https://helm.sh/docs/chart_template_guide/accessing_files/) — встроить файл в ConfigMap
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) — the default StorageClass in k3d
  * [local-path-provisioner](https://github.com/rancher/local-path-provisioner) — StorageClass по умолчанию в k3d

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs — read these BEFORE you debug)</summary>

⚠️ Частые грабли (из реальных прогонов — прочитай до того, как начнёшь отлаживать)

- **ConfigMap env vars don't refresh in a running pod.** Change `LOG_LEVEL` in the CM, `printenv` in the running pod still shows the old value — forever, until the pod restarts. This is the #1 "my ConfigMap update isn't working" cause. Env vars are baked at pod start; nothing changes that.
  * **Переменные окружения из ConfigMap не обновляются в работающем поде.** Поменяй `LOG_LEVEL` в ConfigMap — `printenv` в работающем поде всё равно покажет старое значение, и так до перезапуска пода. Это причина №1 жалоб «мой ConfigMap не обновляется». Переменные окружения фиксируются при старте пода, и это никак не обойти.
- **`subPath` mounts NEVER auto-update.** A `subPath: config.json` mount is a one-shot file copy at pod start. The ConfigMap can change every five seconds; the file in the container won't. Whole-directory mounts DO auto-update (within `kubelet --sync-frequency`, default 60s + cache TTL — total delay can be a couple of minutes). If you want hot reload, mount the whole dir.
  * **Монтирования через `subPath` НИКОГДА не обновляются автоматически.** `subPath: config.json` — это разовая копия файла при старте пода. ConfigMap может меняться хоть каждые пять секунд, файл в контейнере — нет. Монтирование всего каталога обновляется автоматически (в пределах `kubelet --sync-frequency`, по умолчанию 60 с + TTL кэша, так что суммарная задержка может достигать пары минут). Хочешь горячую перезагрузку — монтируй весь каталог.
- **`WaitForFirstConsumer` PVC stays `Pending` — that's normal, not an error.** The default StorageClass on k3d (and on most cloud SCs in multi-AZ clusters) waits to bind until a pod actually mounts the PVC, so the PV can be provisioned in the same zone as the pod. If your PVC is `Pending` and no pod has consumed it yet, just deploy the pod — don't try to "fix" the SC.
  * **PVC при `WaitForFirstConsumer` остаётся в `Pending` — это нормально, а не ошибка.** StorageClass по умолчанию в k3d (как и большинство облачных классов в мультизональных кластерах) ждёт с привязкой, пока под реально не смонтирует PVC, чтобы том создался в той же зоне, что и под. Если PVC в `Pending` и под ещё его не использовал, просто задеплой под — не пытайся «чинить» StorageClass.
- **`kubectl exec ... Unknown stream id` is harmless.** Client-side websocket message from a benign control-frame mismatch. The data round-trips fine. Don't chase it.
  * **`kubectl exec ... Unknown stream id` безобиден.** Это клиентское сообщение websocket из-за незначительного расхождения control-фреймов. Данные передаются туда и обратно нормально. Не гоняйся за ним.
- **Deleting a PVC may NOT delete the PV (or the data).** Depends on `reclaimPolicy`: `Delete` (default for cloud SCs) wipes the volume; `Retain` keeps the PV and the disk after the PVC is gone — you must clean up manually. Set `Retain` for anything you'd cry over losing, and write the cleanup runbook **before** the day you need it.
  * **Удаление PVC может НЕ удалить PV (и данные).** Всё зависит от `reclaimPolicy`: `Delete` (по умолчанию для облачных классов) стирает том; `Retain` сохраняет PV и диск после удаления PVC — чистить придётся вручную. Ставь `Retain` для всего, что не хотел бы потерять, и напиши инструкцию по очистке **до** того дня, когда она понадобится.
- **Invalid env-var names from `envFrom` are silently skipped.** A ConfigMap key `feature.x.enabled` won't appear as an env var — `envFrom` only emits keys matching `[A-Z_][A-Z0-9_]*`. No error, no warning, just missing.
  * **Некорректные имена переменных при `envFrom` молча пропускаются.** Ключ ConfigMap `feature.x.enabled` не появится как переменная окружения — `envFrom` выдаёт только ключи, подходящие под `[A-Z_][A-Z0-9_]*`. Ни ошибки, ни предупреждения — просто отсутствие.
- **`hostPath` for "real" persistence ties a pod to a node.** Demo only. Use a PVC even for local clusters — the local-path provisioner gives you a real PVC backed by a host directory, with proper rescheduling semantics.
  * **`hostPath` для «настоящей» сохранности привязывает под к узлу.** Годится только для демо. Даже в локальных кластерах используй PVC: local-path provisioner даёт настоящий PVC поверх каталога хоста с корректной семантикой перепланирования.
- **Don't overwrite `/data/visits` with your marker.** Use a *separate* file (e.g. `/data/marker`) for the 3.5 persistence proof. The visit counter is rewriting `visits` on every `GET /`, which makes side-by-side comparison flaky.
  * **Не перезаписывай `/data/visits` своим маркером.** Для доказательства из 3.5 бери *отдельный* файл (например, `/data/marker`). Счётчик посещений перезаписывает `visits` при каждом `GET /`, из-за чего сравнение «рядом» становится ненадёжным.

</details>

<details>
<summary>🛠️ Useful CLI (no YAML — exec recipes only)</summary>

🛠️ Полезный CLI (без YAML — только рецепты exec)

- `kubectl get sc` / `kubectl describe sc <name>` — what storage the cluster offers
  * `kubectl get sc` / `kubectl describe sc <name>` — какое хранилище предлагает кластер
- `kubectl describe pvc <name>` — check Events at the bottom when a PVC is stuck
  * `kubectl describe pvc <name>` — смотри Events в конце вывода, если PVC застрял
- `kubectl describe pod <name> | grep -A4 'Mounts\|Volumes'` — what's actually mounted
  * `kubectl describe pod <name> | grep -A4 'Mounts\|Volumes'` — что реально смонтировано
- `kubectl exec <pod> -- mount | grep /data` — confirm the volume is mounted
  * `kubectl exec <pod> -- mount | grep /data` — убедиться, что том смонтирован
- `kubectl get events --field-selector involvedObject.name=<pvc>` — provisioning errors
  * `kubectl get events --field-selector involvedObject.name=<pvc>` — ошибки создания тома
- For the persistence proof: write a marker with `kubectl exec deploy/<name> -- sh -c 'echo $(date +%s) > /data/marker'` (NOT `/data/visits` — the counter overwrites that), then `kubectl delete pod -l <selector>`, then `kubectl wait --for=condition=Ready pod -l <selector> --timeout=60s`, then `kubectl exec deploy/<name> -- cat /data/marker`
  * Для доказательства сохранности: запиши маркер командой `kubectl exec deploy/<name> -- sh -c 'echo $(date +%s) > /data/marker'` (НЕ в `/data/visits` — счётчик его перезапишет), затем `kubectl delete pod -l <selector>`, затем `kubectl wait --for=condition=Ready pod -l <selector> --timeout=60s`, затем `kubectl exec deploy/<name> -- cat /data/marker`

</details>

---

## Looking Ahead

Что дальше

| Lab | What it adds |
|---:|---|
| 13 | ArgoCD GitOps — your ConfigMap + PVC reconciled from git |
| 14 | Argo Rollouts — canary the new ConfigMap before all pods see it |
| 15 | StatefulSets + `volumeClaimTemplates` — one PVC *per* replica, for Postgres/Kafka |
| 16 | kube-prometheus-stack — scrape `/metrics` from your app, alert on visit-count regressions |

| Лаба | Что добавляет |
|---:|---|
| 13 | ArgoCD GitOps — твои ConfigMap и PVC синхронизируются из git |
| 14 | Argo Rollouts — канареечный выкат нового ConfigMap, прежде чем его увидят все поды |
| 15 | StatefulSets + `volumeClaimTemplates` — один PVC *на* реплику, для Postgres/Kafka |
| 16 | kube-prometheus-stack — сбор `/metrics` из твоего приложения, алерты на регрессии счётчика посещений |

> **Remember:** the goal of the lab is not "PVC `Bound`". The goal is "the value I wrote in pod A came back from pod B after I deleted pod A." That's persistence. Everything else is plumbing.
>
> **Помни:** цель лабы — не «PVC в статусе `Bound`». Цель — «значение, которое я записал в под A, вернулось из пода B после того, как я удалил под A». Вот это и есть сохранность. Всё остальное — слесарная работа.
