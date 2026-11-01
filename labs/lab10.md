# Lab 10 — Helm Package Manager

Лабораторная 10 — Пакетный менеджер Helm

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Helm-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Helm%204-informational)

> **Goal:** Convert your Lab 9 Kubernetes manifests into a hand-written Helm 4 chart that ships through dev / staging / prod with one value file per environment, runs a lifecycle hook, and (bonus) is publishable to GHCR over OCI.
> **Deliverable:** A PR from `lab10` adding `k8s/lab10-app/` (chart) and `k8s/lab10-app/HELM.md` (your report).
>
> **Цель:** превратить манифесты Kubernetes из лабы 9 в написанный вручную Helm 4-чарт, который доставляется через dev / staging / prod с отдельным файлом значений на каждое окружение, запускает хук жизненного цикла и (в бонусной части) публикуется в GHCR через OCI.
> **Сдача:** PR из ветки `lab10`, добавляющий `k8s/lab10-app/` (чарт) и `k8s/lab10-app/HELM.md` (твой отчёт).

---

## Overview

Обзор

In Lab 9 you wrote four raw manifests (web Deployment + Service, echo Deployment + Service). For one environment that's fine — for three, you'd be copy-editing YAML and praying nothing drifts.

В лабе 9 ты написал четыре «сырых» манифеста (Deployment + Service для web и Deployment + Service для echo). Для одного окружения это нормально — а для трёх пришлось бы копировать и править YAML, молясь, чтобы ничего не разъехалось.

This lab pulls every hardcoded value out of those manifests into a **chart** with a **values file per environment**. The skill being graded is **template authoring**: writing `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`, `templates/service.yaml`, `templates/_helpers.tpl`, and one hook — from scratch. Not `helm create`-and-edit. From scratch.

Эта лаба выносит все захардкоженные значения из тех манифестов в **чарт** с **файлом значений на каждое окружение**. Оценивается навык **написания шаблонов**: `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`, `templates/service.yaml`, `templates/_helpers.tpl` и один хук — с нуля. Не «`helm create` и поправь». С нуля.

> ⚠️ **Scope:** No subcharts, no library charts, no `values.schema.json`. Just chart anatomy + values hierarchy + one hook + (bonus) an OCI push.
>
> ⚠️ **Границы работы:** никаких сабчартов, библиотечных чартов и `values.schema.json`. Только анатомия чарта + иерархия значений + один хук + (бонус) пуш по OCI.

**What you'll practice:**

**Что ты отработаешь на практике:**

- Helm 4 chart anatomy: `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`
  * Анатомия Helm 4-чарта: `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`
- Go templates + Sprig (`include`, `nindent`, `default`, `toYaml`, `printf`, `trunc`, `quote`)
  * Go-шаблоны + Sprig (`include`, `nindent`, `default`, `toYaml`, `printf`, `trunc`, `quote`)
- The release lifecycle: `install / upgrade / rollback / list / history / uninstall`
  * Жизненный цикл релиза: `install / upgrade / rollback / list / history / uninstall`
- Multi-environment delivery via a values hierarchy and **`-f a -f b`** layering
  * Доставка в несколько окружений через иерархию значений и наложение **`-f a -f b`**
- Lifecycle hooks (`pre-install` / `pre-upgrade`) with weights and delete policies
  * Хуки жизненного цикла (`pre-install` / `pre-upgrade`) с весами и политиками удаления
- Publishing charts as OCI artifacts to GHCR
  * Публикация чартов как OCI-артефактов в GHCR

> 📚 Pairs with **Lecture 10 — Helm Package Management**. Re-read slides 5–11 (chart anatomy, templates, `_helpers.tpl`, lifecycle, multi-env, hooks) before you start.
>
> 📚 В паре с **лекцией 10 — Управление пакетами в Helm**. Перед началом перечитай слайды 5–11 (анатомия чарта, шаблоны, `_helpers.tpl`, жизненный цикл, несколько окружений, хуки).

---

## Project State

Состояние проекта

**You should have from previous labs:**

**У тебя должно быть с прошлых лаб:**

- A k3d cluster on Kubernetes 1.36 (Lab 9 — `k3d cluster create devops ...`)
  * Кластер k3d на Kubernetes 1.36 (лаба 9 — `k3d cluster create devops ...`)
- Your **Lab 2 web image** in a public registry (Docker Hub / GHCR)
  * **Образ web-приложения из лабы 2** в публичном реестре (Docker Hub / GHCR)
- Lab 9 manifests in `k8s/`: `web-deployment.yaml`, `web-service.yaml`, `echo-deployment.yaml`, `echo-service.yaml`
  * Манифесты лабы 9 в `k8s/`: `web-deployment.yaml`, `web-service.yaml`, `echo-deployment.yaml`, `echo-service.yaml`

**This lab adds:**

**Эта лаба добавляет:**

- `k8s/lab10-app/` — a hand-written Helm 4 chart that **replaces** those four manifests
  * `k8s/lab10-app/` — написанный вручную Helm 4-чарт, который **заменяет** эти четыре манифеста
- `k8s/lab10-app/values-prod.yaml` (and optionally `-dev`, `-staging`) — per-environment deltas
  * `k8s/lab10-app/values-prod.yaml` (и по желанию `-dev`, `-staging`) — отличия по окружениям
- `k8s/lab10-app/templates/hooks/db-migrate.yaml` — one pre-install/pre-upgrade hook
  * `k8s/lab10-app/templates/hooks/db-migrate.yaml` — один хук `pre-install`/`pre-upgrade`
- `k8s/lab10-app/HELM.md` — your submission report
  * `k8s/lab10-app/HELM.md` — отчёт о сдаче работы

By Lab 13 ArgoCD will deploy this chart through GitOps; by Lab 14 Argo Rollouts will replace the Deployment template you write here. So the template hygiene you build this week outlives Lab 10.

К лабе 13 ArgoCD будет разворачивать этот чарт через GitOps; к лабе 14 Argo Rollouts заменит написанный здесь шаблон Deployment. Так что аккуратность шаблонов, которую ты выработаешь на этой неделе, переживёт лабу 10.

---

## Setup

Настройка

```bash
helm version            # must report v4.1.x (course pins to 4.1.4)
kubectl get nodes       # your Lab 9 cluster must be up
```

Пояснение: первая команда проверяет версию Helm (нужна v4.1.x), вторая — что кластер из лабы 9 запущен и узлы доступны.

> **Helm 3 vs Helm 4:** Helm 3 is in support-mode only (bug fixes through July 8 2026, security fixes through November 11 2026). The course standardizes on **Helm 4**. Note that `apiVersion: v2` in `Chart.yaml` is still correct on Helm 4 — that header is the Helm-3-and-later chart-format version, **not** the Helm binary version. `apiVersion: v1` is Helm 2 only.
>
> **Helm 3 против Helm 4:** Helm 3 находится только в режиме поддержки (исправления ошибок до 8 июля 2026 года, исправления безопасности до 11 ноября 2026 года). Курс стандартизирован на **Helm 4**. Обрати внимание: `apiVersion: v2` в `Chart.yaml` по-прежнему корректен для Helm 4 — этот заголовок задаёт версию формата чарта (Helm 3 и новее), а **не** версию бинарника Helm. `apiVersion: v1` — только для Helm 2.

Install instructions: <https://helm.sh/docs/intro/install/>.

Инструкции по установке: <https://helm.sh/docs/intro/install/>.

You will write every file under `k8s/lab10-app/` yourself — **do not run `helm create`** (see Common Pitfalls).

Все файлы в `k8s/lab10-app/` ты напишешь сам — **не запускай `helm create`** (см. раздел «Типичные ошибки»).

---

## Task 1 — Helm fundamentals & setup (1 pt)

Задание 1 — Основы Helm и настройка (1 балл)

### 1.1 — Install Helm 4 and inspect a real chart

1.1 — Установить Helm 4 и изучить реальный чарт

`YOUR TASK`: install (or upgrade to) Helm 4.1.x, then pull metadata and default values from an OCI-hosted public chart **without installing it**:

`YOUR TASK`: установи (или обнови до) Helm 4.1.x, затем получи метаданные и значения по умолчанию из публичного чарта, размещённого в OCI, **не устанавливая его**:

```bash
helm show chart  oci://registry-1.docker.io/bitnamicharts/nginx
helm show values oci://registry-1.docker.io/bitnamicharts/nginx | head -40
```

Пояснение: `helm show chart` выводит метаданные чарта (содержимое `Chart.yaml`), `helm show values` — его значения по умолчанию; `head -40` обрезает вывод до первых сорока строк. Установка при этом не выполняется.

Look at the shape: `apiVersion`, `name`, `version`, `appVersion`, `dependencies`; in `values.yaml` notice how every numeric/string knob is exposed.

Посмотри на структуру: `apiVersion`, `name`, `version`, `appVersion`, `dependencies`; в `values.yaml` обрати внимание, как вынесен наружу каждый числовой и строковый параметр.

### 1.2 — Explain the three concepts in your own words

1.2 — Объяснить три концепции своими словами

In `HELM.md`, write 2–3 sentences each for **Chart**, **Release**, **Values**. Don't copy the docs — you must be able to answer them at a viva. Hints:

В `HELM.md` напиши по 2–3 предложения для **Chart**, **Release** и **Values**. Не копируй документацию — ты должен уметь ответить на защите. Подсказки:

- *Chart* is the package on disk; *Release* is one running instance of it; *Values* fill the template.
  * *Chart* — это пакет на диске; *Release* — один запущенный экземпляр этого пакета; *Values* заполняют шаблон.
- Helm 4 stores release state in a `helm.sh/release.v1` Secret in the release's namespace (Tiller is long gone — Helm 2).
  * Helm 4 хранит состояние релиза в Secret `helm.sh/release.v1` в пространстве имён релиза (Tiller давно в прошлом — это было в Helm 2).

### 1.3 — Proof of work

1.3 — Подтверждение работы

Paste into `HELM.md`:

Вставь в `HELM.md`:

- `helm version` output showing v4.1.x
  * вывод `helm version`, показывающий v4.1.x
- The first ~20 lines of one of the `helm show` calls above
  * первые ~20 строк одного из вызовов `helm show` выше
- Your 2–3-sentence explanations of Chart vs Release vs Values
  * свои объяснения Chart vs Release vs Values в 2–3 предложениях

---

## Task 2 — Build the chart (3 pts)

Задание 2 — Собрать чарт (3 балла)

**Objective:** Write the chart from scratch. The skeleton below is the **directory shape** you must produce. **Do not** run `helm create`; you'd ship boilerplate references to a `serviceaccount.yaml` you didn't write, then break `helm lint` the moment you edit `values.yaml` (see Common Pitfalls).

**Цель:** написать чарт с нуля. Скелет ниже — это **структура каталогов**, которую ты должен получить. **Не** запускай `helm create`: ты получишь шаблонный код со ссылками на `serviceaccount.yaml`, которого сам не писал, и `helm lint` сломается в тот же момент, когда ты отредактируешь `values.yaml` (см. раздел «Типичные ошибки»).

```
k8s/lab10-app/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml                # Task 3
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml             # web
    ├── service.yaml                # web
    ├── echo-deployment.yaml        # echo (Task 2.6)
    ├── echo-service.yaml           # echo (Task 2.6)
    ├── hooks/
    │   └── db-migrate.yaml         # Task 4
    └── NOTES.txt                   # optional; printed after install
```

Пояснение: это дерево файлов чарта. Комментарии рядом с файлами показывают, к какому заданию они относятся.

### 2.1 — `Chart.yaml`

2.1 — Файл `Chart.yaml`

`YOUR TASK`: fill in the metadata. Both `version` (chart SemVer — bump on chart changes) and `appVersion` (image-side version string — bump on image changes) matter; they evolve independently. `kubeVersion` makes Helm refuse to install against an older cluster.

`YOUR TASK`: заполни метаданные. Важны и `version` (SemVer чарта — повышается при изменениях чарта), и `appVersion` (строка версии на стороне образа — повышается при изменениях образа); они развиваются независимо. `kubeVersion` заставляет Helm отказаться от установки в более старый кластер.

```yaml
apiVersion: ___                  # YOUR TASK: chart-format version (Helm 3/4 expects this value)
name: ___                        # YOUR TASK: chart name (must match the directory name)
description: ___                 # YOUR TASK: one sentence; shows in `helm show chart`
type: application                # 'application' (default) or 'library' — leave as-is for this lab
version: ___                     # YOUR TASK: SemVer for the chart, e.g. 0.1.0
appVersion: "___"                # YOUR TASK: quoted string, image-side version (your Lab 2 tag)
kubeVersion: "___"               # YOUR TASK: a range, e.g. ">=1.33.0" — refuses older clusters
# dependencies:                  # OPTIONAL — not used in this lab
#   - name: ...
#     version: ...
#     repository: ...
```

Пояснение: вместо каждого `___` нужно подставить своё значение; комментарии `YOUR TASK` в коде объясняют, что именно требуется. Комментарии через `#` — это обычные YAML-комментарии, они остаются как есть.

### 2.2 — `values.yaml`

2.2 — Файл `values.yaml`

`YOUR TASK`: list every value your templates will reference, with a sensible default. **Every hardcoded number/string from your Lab 9 manifests becomes a value here.** Both `web` and `echo` need their own blocks.

`YOUR TASK`: перечисли все значения, на которые будут ссылаться твои шаблоны, с разумным значением по умолчанию. **Каждое захардкоженное число и строка из манифестов лабы 9 становятся здесь значением.** Собственные блоки нужны и для `web`, и для `echo`.

The minimum surface (you may add more):

Минимальный набор (можно добавить больше):

```yaml
# YOUR TASK: fill every default. The keys below are the contract between
# values.yaml and your templates — your templates will fail to render if you
# rename a key here without updating the template, and vice versa.

replicaCount: ___                # default replicas for `web`

image:
  repository: ___                # your Lab 2 image (GHCR or Docker Hub), e.g. ghcr.io/<you>/devops-info-service
  tag: ""                        # empty → templates fall back to .Chart.AppVersion
  pullPolicy: ___                # IfNotPresent | Always | Never (Lab 9's pitfalls call out the k3d-import case)

service:
  type: ___                      # NodePort for local k3d; LoadBalancer in prod
  port: ___                      # the Service port (clients hit this)
  # the container's listen port lives under `containerPort` below

containerPort: ___               # what your Lab 2 app listens on (Lab 9 set this — match it)

resources:
  requests:
    cpu: ___
    memory: ___
  limits:
    cpu: ___
    memory: ___

probes:
  liveness:
    path: ___                    # /health from Lab 1
    initialDelaySeconds: ___
    periodSeconds: ___
  readiness:
    path: ___
    periodSeconds: ___

ingress:
  enabled: false                 # not used in this lab; reserved for Lab 16

echo:
  replicaCount: ___              # default replicas for the echo sidecar service (Lab 9 had a specific count)
  image:
    repository: ghcr.io/inno-devops-labs/echo
    tag: v1
  service:
    type: ClusterIP
    port: 80
    targetPort: 8081
  resources:                     # echo is a tiny Go binary — size it smaller than web (Lab 9 §3.1)
    requests:
      cpu: ___
      memory: ___
    limits:
      cpu: ___
      memory: ___

# Hook config (Task 4)
hooks:
  dbMigrate:
    enabled: true
    image: busybox:1.37          # any small image — the hook is a fake migration
```

Пояснение: ключи в этом файле — контракт между `values.yaml` и шаблонами: если переименуешь ключ здесь и не обновишь шаблон (или наоборот), рендеринг упадёт. В блоке `echo` часть значений уже задана, а `___` нужно заполнить самому.

### 2.3 — `templates/_helpers.tpl`

2.3 — Хелперы и именованные шаблоны (`templates/_helpers.tpl`)

Named templates avoid copy-pasting label blocks across every manifest. You will define **four**: `name`, `fullname`, `labels`, `selectorLabels`. They are the muscle memory of Helm; you'll write them in every chart for the rest of your career.

Именованные шаблоны избавляют от копирования блоков меток в каждый манифест. Ты определишь **четыре**: `name`, `fullname`, `labels`, `selectorLabels`. Это мышечная память Helm — ты будешь писать их в каждом чарте всю карьеру.

`YOUR TASK`: implement the bodies. The function shape is given — fill the body of each `define` block so that:

`YOUR TASK`: реализуй тела. Форма функции дана — заполни тело каждого блока `define` так, чтобы:

| Template | What it must return |
|---|---|
| `lab10-app.name` | The chart name (or `nameOverride` if a user sets one), truncated to 63 chars (K8s label limit) and with any trailing `-` stripped. |
| `lab10-app.fullname` | A unique-per-release name: `<release-name>-<chart-name>` (or `nameOverride` if set), truncated to 63, trailing `-` stripped. This is what most metadata.name fields use. |
| `lab10-app.labels` | The full label set: `helm.sh/chart`, every selector label (include `selectorLabels`), `app.kubernetes.io/version`, `app.kubernetes.io/managed-by`. |
| `lab10-app.selectorLabels` | A **strict subset** of `labels`: only `app.kubernetes.io/name` and `app.kubernetes.io/instance`. Selectors are immutable after creation — never put `version` here (see Common Pitfalls). |

| Шаблон | Что он должен возвращать |
|---|---|
| `lab10-app.name` | Имя чарта (или `nameOverride`, если пользователь его задал), обрезанное до 63 символов (ограничение меток K8s), с удалённым хвостовым `-`. |
| `lab10-app.fullname` | Уникальное для релиза имя: `<release-name>-<chart-name>` (или `nameOverride`, если задан), обрезанное до 63, хвостовой `-` удалён. Именно его использует большинство полей `metadata.name`. |
| `lab10-app.labels` | Полный набор меток: `helm.sh/chart`, все селекторные метки (включая `selectorLabels`), `app.kubernetes.io/version`, `app.kubernetes.io/managed-by`. |
| `lab10-app.selectorLabels` | **Строгое подмножество** `labels`: только `app.kubernetes.io/name` и `app.kubernetes.io/instance`. Селекторы неизменяемы после создания — никогда не клади сюда `version` (см. раздел «Типичные ошибки»). |

```handlebars
{{/* YOUR TASK: return the chart name, truncated to 63 chars, no trailing dash */}}
{{- define "lab10-app.name" -}}
{{- /* your body — hint: `default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-"` is the canonical form */ -}}
{{- end }}

{{/* YOUR TASK: return a unique-per-release name, truncated to 63, no trailing dash.
     IMPORTANT: when the release name already contains the chart name (e.g. you ran
     `helm install lab10-app k8s/lab10-app`), the canonical helper collapses to JUST
     the release name — otherwise you'd get `lab10-app-lab10-app`. Lab 11 references
     `deploy/lab10-app-web`, so your fullname helper MUST handle the collapse. The
     `helm create` boilerplate has the canonical form; cribbing the pattern (not the
     whole file) is fair game.
     Shape: `if contains $name .Release.Name → .Release.Name`, else
            `printf "%s-%s" .Release.Name $name`, then `| trunc 63 | trimSuffix "-"` */}}
{{- define "lab10-app.fullname" -}}
{{- /* your body */ -}}
{{- end }}

{{/* YOUR TASK: return the full label set (5 labels including the helm.sh/chart line) */}}
{{- define "lab10-app.labels" -}}
{{- /* your body — must include `lab10-app.selectorLabels`; emit `helm.sh/chart`,
       `app.kubernetes.io/version`, `app.kubernetes.io/managed-by` */ -}}
{{- end }}

{{/* YOUR TASK: return ONLY the two selector labels (name + instance). No version. */}}
{{- define "lab10-app.selectorLabels" -}}
{{- /* your body */ -}}
{{- end }}
```

Пояснение: `define` объявляет именованный шаблон, `end` его закрывает. Конструкция `{{- ... -}}` убирает лишние пробелы и переводы строк вокруг действия, а `include` вызывает один шаблон из другого. Комментарии `{{/* ... */}}` — подсказки о том, что нужно написать.

Reference functions you'll need (all from Sprig): `default`, `trunc`, `trimSuffix`, `printf`, `replace`, `quote`. See [Sprig docs](https://masterminds.github.io/sprig/) and [Chart Template Guide](https://helm.sh/docs/chart_template_guide/).

Функции, которые понадобятся (все из Sprig): `default`, `trunc`, `trimSuffix`, `printf`, `replace`, `quote`. См. [документацию Sprig](https://masterminds.github.io/sprig/) и [руководство по шаблонам чартов](https://helm.sh/docs/chart_template_guide/).

### 2.4 — `templates/deployment.yaml` (web)

2.4 — Шаблон деплоймента web (`templates/deployment.yaml`)

`YOUR TASK`: templatize your Lab 9 web Deployment. The YAML shape is given so you don't reinvent the wheel — each `{{ ___ }}` is a token you must fill (and many of those will need pipelines like `| nindent 4` or `| default ...`).

`YOUR TASK`: преврати Deployment web из лабы 9 в шаблон. Форма YAML дана, чтобы не изобретать велосипед — каждый `{{ ___ }}` нужно заполнить (и многие потребуют конвейеров вроде `| nindent 4` или `| default ...`).

> 💡 **Name web and echo distinctly.** Both Deployments are in the same release; if `metadata.name` resolves to the same value on both, the second `helm install` errors out with a conflict. Convention: append a component suffix to the fullname (e.g. `{{ include "lab10-app.fullname" . }}-web` here, `-echo` on the echo manifests in 2.6). Lab 11 will reference `deploy/lab10-app-web` directly — keep the suffix.
>
> 💡 **Называй web и echo по-разному.** Оба Deployment входят в один релиз; если `metadata.name` у них совпадёт, второй `helm install` завершится ошибкой конфликта. Соглашение: добавляй к fullname суффикс компонента (например, `{{ include "lab10-app.fullname" . }}-web` здесь и `-echo` в манифестах echo в 2.6). Лаба 11 будет обращаться к `deploy/lab10-app-web` напрямую — сохрани суффикс.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ ___ }}-web                            # YOUR TASK: `include` your fullname template (suffix already present)
  labels:
    {{- ___ | nindent 4 }}                       # YOUR TASK: `include` your labels template
spec:
  replicas: {{ ___ }}                            # YOUR TASK: from .Values.replicaCount (with a sane default)
  selector:
    matchLabels:
      {{- ___ | nindent 6 }}                     # YOUR TASK: `include` your SELECTOR labels template
  template:
    metadata:
      labels:
        {{- ___ | nindent 8 }}                   # YOUR TASK: `include` your SELECTOR labels template
    spec:
      containers:
        - name: web
          image: "{{ ___ }}:{{ ___ | default .Chart.AppVersion }}"   # YOUR TASK: repo + tag from values
          imagePullPolicy: {{ ___ }}             # YOUR TASK: from .Values.image.pullPolicy
          ports:
            - name: http
              containerPort: {{ ___ }}           # YOUR TASK: from .Values.containerPort
              protocol: TCP
          livenessProbe:
            httpGet:
              path: {{ ___ }}                    # YOUR TASK: from values
              port: http
            initialDelaySeconds: {{ ___ }}
            periodSeconds: {{ ___ }}
          readinessProbe:
            httpGet:
              path: {{ ___ }}
              port: http
            periodSeconds: {{ ___ }}
          {{- with .Values.resources }}
          resources:
            {{- ___ | nindent 12 }}              # YOUR TASK: render the whole sub-tree as YAML (one Sprig fn)
          {{- end }}
```

Пояснение: `{{ ___ }}` — места, которые нужно заполнить шаблонными действиями; комментарии `# YOUR TASK` подсказывают, что подставить. Отступы внутри блоков `nindent` должны совпадать с уровнем YAML-ключа.

> **`indent` vs `nindent`:** `nindent N` = newline + indent by N spaces. `indent N` = indent only. When a `{{- ... -}}` action sits on its own line and consumed the preceding newline, `indent` produces invalid YAML. Use `nindent` 9 times out of 10.
>
> **`indent` против `nindent`:** `nindent N` = перевод строки + отступ в N пробелов. `indent N` = только отступ. Когда действие `{{- ... -}}` стоит на отдельной строке и «съело» предыдущий перевод строки, `indent` даёт невалидный YAML. В девяти случаях из десяти используй `nindent`.

### 2.5 — `templates/service.yaml` (web)

2.5 — Шаблон сервиса web (`templates/service.yaml`)

`YOUR TASK`: same drill, much shorter.

`YOUR TASK`: то же самое, но намного короче.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ ___ }}-web                            # YOUR TASK: fullname template (`-web` suffix mirrors the Deployment)
  labels:
    {{- ___ | nindent 4 }}                       # YOUR TASK: labels template
spec:
  type: {{ ___ }}                                # YOUR TASK: from .Values.service.type
  ports:
    - port: {{ ___ }}                            # YOUR TASK: from .Values.service.port
      targetPort: http                           # named ports beat numbers; this points at the deployment's `name: http`
      protocol: TCP
      name: http
  selector:
    {{- ___ | nindent 4 }}                       # YOUR TASK: SELECTOR labels template (NOT the full labels)
```

Пояснение: обрати внимание на `targetPort: http` — именованные порты надёжнее числовых, здесь он указывает на `name: http` из деплоймента. В `selector` используются только селекторные метки, а не полный набор меток.

### 2.6 — `templates/echo-deployment.yaml` + `templates/echo-service.yaml`

2.6 — Манифесты echo (`templates/echo-deployment.yaml` + `templates/echo-service.yaml`)

`YOUR TASK`: same pattern, but for the echo sidecar. Two notable differences:

`YOUR TASK`: тот же подход, но для sidecar-сервиса echo. Два заметных отличия:

1. Image comes from `.Values.echo.image.repository` + `.Values.echo.image.tag` (no `default .Chart.AppVersion` — echo's image is independent of your app's `appVersion`).
   * Образ берётся из `.Values.echo.image.repository` + `.Values.echo.image.tag` (без `default .Chart.AppVersion` — образ echo не зависит от `appVersion` твоего приложения).
2. Use a different label discriminator (e.g. `app.kubernetes.io/component: echo`) on the pod template + Service selector so the **web** Service doesn't accidentally select echo pods. You can either:
   * Используй другой дискриминатор меток (например, `app.kubernetes.io/component: echo`) в шаблоне пода и селекторе Service, чтобы Service **web** случайно не выбрал поды echo. Можно либо:
   - Extend your `selectorLabels` helper with an optional component (pass `.Values.echo` into `include`), **or**
     * расширить хелпер `selectorLabels` необязательным компонентом (передавая `.Values.echo` в `include`), **либо**
   - Hand-write a small additional selector key in this manifest only.
     * вручную добавить небольшой дополнительный ключ селектора только в этом манифесте.

Take a moment to decide which — both are valid.

Удели минуту и реши, какой вариант выбрать — оба допустимы.

### 2.7 — Validate, render, install

2.7 — Проверить, отрендерить, установить

Run, in this order: `helm lint <chart>`, `helm template <release> <chart>` (eyeball the rendered YAML), `helm install <release> <chart> --dry-run --debug`, then a real `helm install --create-namespace` into a fresh namespace. Finish with `helm list` (expect `REVISION 1`) and `kubectl get deploy,svc,pods` to confirm both pods are Ready.

Выполни по порядку: `helm lint <chart>`, `helm template <release> <chart>` (просмотри отрендеренный YAML), `helm install <release> <chart> --dry-run --debug`, а затем настоящий `helm install --create-namespace` в новое пространство имён. Заверши проверкой `helm list` (ожидается `REVISION 1`) и `kubectl get deploy,svc,pods`, чтобы убедиться, что оба пода в состоянии Ready.

```bash
# Release name MUST be `lab10-app` (matches chart name). Lab 11 references `deploy/lab10-app-web`;
# if your fullname helper has the canonical collapse-when-contained branch (see 2.3), this works.
# If you used a too-simple fullname (just `printf "%s-%s" .Release.Name .Chart.Name`), you'll get
# `lab10-app-lab10-app-web` and Lab 11 won't find your Deployment. Run `helm template` first to verify.
helm install lab10-app k8s/lab10-app -n helm-demo --create-namespace
# expected: helm list shows STATUS deployed, REVISION 1
# expected (verify with `kubectl get deploy -n helm-demo`): deploy/lab10-app-web and deploy/lab10-app-echo
```

Пояснение: имя релиза обязано быть `lab10-app` (совпадает с именем чарта). Если хелпер fullname реализован правильно, Deployment получит имя `lab10-app-web`; при наивной реализации выйдет `lab10-app-lab10-app-web`, и лаба 11 не найдёт твой Deployment. Перед установкой проверь результат через `helm template`.

### 2.8 — Proof of work

2.8 — Подтверждение работы

Paste into `HELM.md`:

Вставь в `HELM.md`:

- `helm lint k8s/lab10-app` output (must say `0 chart(s) failed`)
  * вывод `helm lint k8s/lab10-app` (должно быть `0 chart(s) failed`)
- The first ~30 lines of `helm template lab10-app k8s/lab10-app` showing your values flowing into the rendered Deployment (image, replicas, probes, resources)
  * первые ~30 строк вывода `helm template lab10-app k8s/lab10-app`, где видно, как твои значения попадают в отрендеренный Deployment (образ, реплики, пробы, ресурсы)
- `helm list -n helm-demo` showing **REVISION 1** for release `lab10-app`
  * `helm list -n helm-demo` с **REVISION 1** для релиза `lab10-app`
- `kubectl get deploy,svc,pods -n helm-demo` showing both web and echo pods Ready (deploy names: `lab10-app-web`, `lab10-app-echo`)
  * `kubectl get deploy,svc,pods -n helm-demo`, где поды web и echo готовы (имена деплойментов: `lab10-app-web`, `lab10-app-echo`)

---

## Task 3 — Multi-environment values (3 pts)

Задание 3 — Значения для нескольких окружений (3 балла)

**Objective:** Drive ≥ 2 environments from one chart using a base + override values hierarchy.

**Цель:** управлять как минимум двумя окружениями из одного чарта, используя иерархию «база + переопределения».

### 3.1 — Create `values-prod.yaml` (and optionally `values-dev.yaml` / `values-staging.yaml`)

3.1 — Создать `values-prod.yaml` (и при желании `values-dev.yaml` / `values-staging.yaml`)

`YOUR TASK`: write a **deltas-only** override file — only the keys that differ from `values.yaml`. Suggested shape:

`YOUR TASK`: напиши файл переопределений, содержащий **только отличия** от `values.yaml`, — только те ключи, которые отличаются. Предлагаемая форма:

| | dev (defaults in `values.yaml`) | prod (`values-prod.yaml`) |
|---|---|---|
| `replicaCount` | 1 | 4 |
| `image.tag` | `""` (→ `appVersion`) | a pinned SemVer, e.g. `v1.2.0` |
| `service.type` | `NodePort` | `LoadBalancer` |
| `resources.requests.cpu` | `100m` | `500m` |
| `resources.limits.memory` | `256Mi` | `1Gi` |

| | dev (значения по умолчанию из `values.yaml`) | prod (`values-prod.yaml`) |
|---|---|---|
| `replicaCount` | 1 | 4 |
| `image.tag` | `""` (→ `appVersion`) | закреплённый SemVer, например `v1.2.0` |
| `service.type` | `NodePort` | `LoadBalancer` |
| `resources.requests.cpu` | `100m` | `500m` |
| `resources.limits.memory` | `256Mi` | `1Gi` |

For ≥ 3 envs, add `values-staging.yaml` between them.

Для трёх и более окружений добавь между ними `values-staging.yaml`.

### 3.2 — Upgrade an existing release with a layered values hierarchy

3.2 — Обновить существующий релиз с многоуровневой иерархией значений

This is the **headline proof** of Task 3 — the same chart, the same release, going from REVISION 1 to REVISION 2 because `-f` layered a prod override on top of the base.

Это **главное доказательство** для задания 3 — тот же чарт, тот же релиз переходит с REVISION 1 на REVISION 2, потому что `-f` наложил prod-переопределение поверх базы.

```bash
helm upgrade lab10-app k8s/lab10-app -n helm-demo \
  -f k8s/lab10-app/values.yaml \
  -f k8s/lab10-app/values-prod.yaml
# expected: STATUS deployed, REVISION 2
# expected: deploy/lab10-app-web spec.replicas = your prod override (was 1 at REV 1)
```

Пояснение: два `-f` подряд означают, что сначала загружается базовый `values.yaml`, затем `values-prod.yaml`; второй файл переопределяет первый. Ожидается статус deployed и REVISION 2, а `spec.replicas` у деплоймента равен значению из prod-переопределения.

Then run `helm list`, `kubectl get deploy lab10-app-web -n helm-demo -o jsonpath='{.spec.replicas}'`, and `helm history` to capture the evidence.

Затем выполни `helm list`, `kubectl get deploy lab10-app-web -n helm-demo -o jsonpath='{.spec.replicas}'` и `helm history`, чтобы собрать доказательства.

> **Values precedence (lowest → highest):**
> 1. `values.yaml` (chart default, **always loaded** — you don't have to pass it explicitly)
> 2. Each `-f file.yaml` in order — later files override earlier ones
> 3. `--set key=value` on the CLI
>
> So `-f values.yaml -f values-prod.yaml --set replicaCount=10` ends with 10 replicas.
>
> **Приоритет значений (от низшего к высшему):**
> 1. `values.yaml` (значение из чарта, **загружается всегда** — передавать его явно не нужно)
> 2. каждый `-f file.yaml` по порядку — более поздние файлы переопределяют более ранние
> 3. `--set key=value` в командной строке
>
> Значит, `-f values.yaml -f values-prod.yaml --set replicaCount=10` даст в итоге 10 реплик.

### 3.3 — Render a per-env diff without a cluster

3.3 — Отрендерить diff по окружениям без кластера

`YOUR TASK`: use `helm template` against the chart **without** and **with** `-f values-prod.yaml`, then `diff` the two outputs. The visible delta is your proof the override file did something — no cluster required. Capture the first ~20 diff lines in `HELM.md`.

`YOUR TASK`: выполни `helm template` для чарта **без** `-f values-prod.yaml` и **с ним**, затем сравни два вывода командой `diff`. Видимая разница — доказательство, что файл переопределений сработал; кластер не нужен. Перенеси первые ~20 строк diff в `HELM.md`.

### 3.4 — Rollback

3.4 — Откат

`YOUR TASK`: `helm rollback lab10-app 1 -n helm-demo`, then re-query the deployment's replica count — it must return to the dev default. Capture `helm history` showing REVISION 3 with the rollback description.

`YOUR TASK`: выполни `helm rollback lab10-app 1 -n helm-demo`, затем снова запроси число реплик деплоймента — оно должно вернуться к dev-значению по умолчанию. Сохрани `helm history`, где видна REVISION 3 с описанием отката.

### 3.5 — Proof of work

3.5 — Подтверждение работы

Paste into `HELM.md`:

Вставь в `HELM.md`:

- The contents of `values-prod.yaml` (deltas only — should be short)
  * содержимое `values-prod.yaml` (только отличия — файл должен быть коротким)
- `helm list -n helm-demo` **at REVISION 2**, captured after the upgrade
  * `helm list -n helm-demo` **на REVISION 2**, снятый после обновления
- `kubectl get deploy lab10-app-web -n helm-demo -o jsonpath='{.spec.replicas}'` returning the prod value
  * вывод `kubectl get deploy lab10-app-web -n helm-demo -o jsonpath='{.spec.replicas}'` с prod-значением
- `helm history lab10-app -n helm-demo` showing REVISION 1 superseded by REVISION 2
  * `helm history lab10-app -n helm-demo`, где REVISION 1 заменена на REVISION 2
- The first 20 lines of the `diff` from 3.3 — the visible delta between rendered envs
  * первые 20 строк `diff` из пункта 3.3 — видимую разницу между отрендеренными окружениями

---

## Task 4 — Lifecycle hooks (2 pts)

Задание 4 — Хуки жизненного цикла (2 балла)

**Objective:** Add **one** Job hook that runs at `pre-install` AND `pre-upgrade` (e.g. a fake DB migration), ordered by `hook-weight`, cleaned up by `hook-delete-policy`.

**Цель:** добавить **один** хук-Job, который запускается и на `pre-install`, и на `pre-upgrade` (например, фейковую миграцию БД), упорядочивается по `hook-weight` и очищается по `hook-delete-policy`.

### 4.1 — Write `templates/hooks/db-migrate.yaml`

4.1 — Написать `templates/hooks/db-migrate.yaml`

The four annotation keys are the entire hook contract — the rest is a normal `batch/v1` Job. `YOUR TASK`: write the file. The skeleton:

Четыре ключа аннотаций — это весь контракт хука, остальное — обычный Job `batch/v1`. `YOUR TASK`: напиши файл. Скелет:

```yaml
{{- if .Values.hooks.dbMigrate.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ ___ }}-db-migrate                  # YOUR TASK: fullname template + "-db-migrate"
  labels:
    {{- ___ | nindent 4 }}                    # YOUR TASK: labels template
  annotations:
    "helm.sh/hook": ___                       # YOUR TASK: TWO comma-separated points — pre-install,pre-upgrade
    "helm.sh/hook-weight": "___"              # YOUR TASK: a string, NOT a number (Helm quirk). Negative runs earlier.
    "helm.sh/hook-delete-policy": ___         # YOUR TASK: comma-separated policies — before-hook-creation,hook-succeeded
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: db-migrate
          image: {{ ___ }}                    # YOUR TASK: from .Values.hooks.dbMigrate.image
          command: ["sh", "-c"]
          args:
            - |
              echo "running pre-install/pre-upgrade migration for release {{ .Release.Name }}";
              sleep 5;
              echo "migration complete";
{{- end }}
```

Пояснение: `{{- if .Values.hooks.dbMigrate.enabled }}` включает и выключает хук через значения; `command`/`args` имитируют миграцию выводом строк и паузой `sleep 5`. Имя Job собирается из fullname и суффикса `-db-migrate`.

The **four annotation keys** you must set (and what they do):

**Четыре ключа аннотаций**, которые нужно задать (и что они делают):

| Annotation | Required value here | What it does |
|---|---|---|
| `helm.sh/hook` | `pre-install,pre-upgrade` | When to run. Comma-separated for multiple points. |
| `helm.sh/hook-weight` | A quoted string like `"-5"` | Ordering among hooks at the same point. Lower runs first; ties broken alphabetically. **Must be a quoted string** — Helm parses it as a string and converts internally. |
| `helm.sh/hook-delete-policy` | `before-hook-creation,hook-succeeded` | When to remove the hook resource. Without this, failed Jobs pile up forever — `helm uninstall` does NOT delete hook resources. |
| (none — the resource is the Job itself) | — | The hook IS a normal Job, just not part of the regular release. |

| Аннотация | Требуемое значение здесь | Что делает |
|---|---|---|
| `helm.sh/hook` | `pre-install,pre-upgrade` | Когда запускать. Несколько точек — через запятую. |
| `helm.sh/hook-weight` | Строка в кавычках, например `"-5"` | Порядок среди хуков в одной точке. Меньшее значение запускается раньше; при равенстве — по алфавиту. **Должна быть строкой в кавычках** — Helm разбирает её как строку и преобразует внутри. |
| `helm.sh/hook-delete-policy` | `before-hook-creation,hook-succeeded` | Когда удалять ресурс хука. Без этого неудачные Job копятся бесконечно — `helm uninstall` НЕ удаляет ресурсы хуков. |
| (не нужна — ресурс и есть Job) | — | Хук — это обычный Job, просто он не входит в обычный состав релиза. |

### 4.2 — Verify the hook fires on install AND upgrade

4.2 — Убедиться, что хук срабатывает при установке И при обновлении

`YOUR TASK`: uninstall the release for a clean slate, then re-install. During the install, run `kubectl get jobs -n helm-demo` (the hook Job should appear) and `kubectl logs job/<hook-job>` to capture your `running pre-install migration` echo. Then `helm upgrade` with `-f values-prod.yaml` and confirm the **same hook fires again** at `pre-upgrade`. Finally, re-query jobs — `before-hook-creation,hook-succeeded` should have deleted the previous one.

`YOUR TASK`: удали релиз для чистого старта и установи заново. Во время установки выполни `kubectl get jobs -n helm-demo` (Job хука должен появиться) и `kubectl logs job/<hook-job>`, чтобы увидеть свой вывод `running pre-install migration`. Затем выполни `helm upgrade` с `-f values-prod.yaml` и убедись, что **тот же хук срабатывает снова**, уже на `pre-upgrade`. Наконец, снова запроси jobs — политика `before-hook-creation,hook-succeeded` должна была удалить предыдущий Job.

### 4.3 — Proof of work

4.3 — Подтверждение работы

Paste into `HELM.md`:

Вставь в `HELM.md`:

- Your `db-migrate.yaml` annotations block (the four-key contract)
  * блок аннотаций из твоего `db-migrate.yaml` (контракт из четырёх ключей)
- `kubectl get jobs -n helm-demo` captured **during** the install (Job present)
  * `kubectl get jobs -n helm-demo`, снятый **во время** установки (Job присутствует)
- `kubectl logs` from the hook Job showing your "running pre-install migration" echo
  * `kubectl logs` из Job-хука с твоим выводом `running pre-install migration`
- `kubectl get jobs -n helm-demo` captured **after** the policy triggers (Job gone)
  * `kubectl get jobs -n helm-demo`, снятый **после** срабатывания политики (Job исчез)

---

## Task 5 — Documentation (1 pt)

Задание 5 — Документация (1 балл)

`YOUR TASK`: create `k8s/lab10-app/HELM.md` covering, in this order:

`YOUR TASK`: создай `k8s/lab10-app/HELM.md` со следующими разделами в указанном порядке:

1. **Chart overview** — directory tree, what each template renders, the `values.yaml` contract
   * **Обзор чарта** — дерево каталогов, что рендерит каждый шаблон, контракт `values.yaml`
2. **Configuration** — table of the top-level values and how `values-prod.yaml` differs from defaults
   * **Конфигурация** — таблица верхнеуровневых значений и отличия `values-prod.yaml` от значений по умолчанию
3. **Helpers** — your four named templates and one sentence on why `selectorLabels` is a subset of `labels`
   * **Хелперы** — твои четыре именованных шаблона и одно предложение о том, почему `selectorLabels` — подмножество `labels`
4. **Hook** — the four annotations on your `db-migrate` Job and what each one does
   * **Хук** — четыре аннотации твоего Job `db-migrate` и что делает каждая
5. **Operations** — the exact `install`, `upgrade -f a -f b`, `history`, `rollback`, `uninstall` commands you ran
   * **Операции** — точные команды `install`, `upgrade -f a -f b`, `history`, `rollback`, `uninstall`, которые ты выполнял
6. **Evidence** — all the captures from Tasks 1.3 / 2.8 / 3.5 / 4.3
   * **Доказательства** — все результаты и снимки из заданий 1.3 / 2.8 / 3.5 / 4.3
7. **Challenges & learnings** — at least one real one (`indent`/`nindent` bug, immutable selector, `hook-weight` not quoted, etc.)
   * **Трудности и выводы** — хотя бы одна реальная проблема (баг с `indent`/`nindent`, неизменяемый селектор, не взятый в кавычки `hook-weight` и т. п.)

---

## Bonus Task — Publish to GHCR via OCI (2 pts)

Бонусное задание — Публикация в GHCR через OCI (2 балла)

**Objective:** Package the chart, push it to GHCR as an OCI artifact, then **install it from the registry** to prove the round-trip.

**Цель:** упаковать чарт, запушить его в GHCR как OCI-артефакт, а затем **установить его из реестра**, доказав, что цикл замкнулся.

`YOUR TASK`: less hand-holding here.

`YOUR TASK`: здесь подсказок меньше.

1. `helm package` your chart into `lab10-app-<version>.tgz`.
   * упакуй чарт командой `helm package` в `lab10-app-<version>.tgz`.
2. Create a **classic** GitHub PAT (Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token) with the **`write:packages`** scope. Fine-grained tokens don't support OCI pushes yet (May 2026). In CI you'd use the auto-injected `GITHUB_TOKEN` instead; locally use the PAT. **Never paste the PAT in a committed file or on the CLI.**
   * создай **классический** PAT GitHub (Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token) с областью прав **`write:packages`**. Fine-grained-токены пока не поддерживают OCI-пуш (май 2026). В CI вместо него используется автоматически подставляемый `GITHUB_TOKEN`; локально — PAT. **Никогда не вставляй PAT в коммитируемый файл или в командную строку.**
3. `echo $CR_PAT | helm registry login ghcr.io -u <your-username> --password-stdin` — the PAT comes via stdin so it doesn't land in shell history.
   * `echo $CR_PAT | helm registry login ghcr.io -u <your-username> --password-stdin` — PAT передаётся через stdin, поэтому не попадает в историю shell.
4. `helm push <tgz> oci://ghcr.io/<your-username>/charts`.
   * запушь чарт командой `helm push <tgz> oci://ghcr.io/<your-username>/charts`.
5. Make the GHCR package **public** (Your profile → Packages → `charts/lab10-app` → Package settings → Change visibility → Public) so your grader can pull it without credentials.
   * сделай пакет GHCR **публичным** (профиль → Packages → `charts/lab10-app` → Package settings → Change visibility → Public), чтобы проверяющий мог скачать его без учётных данных.
6. In a fresh namespace, `helm install … oci://ghcr.io/<your-username>/charts/lab10-app --version <ver>` — and prove the install came **from the registry**, not your local directory (the `STATUS` will show the OCI source).
   * в новом пространстве имён выполни `helm install … oci://ghcr.io/<your-username>/charts/lab10-app --version <ver>` и докажи, что установка идёт **из реестра**, а не из локального каталога (`STATUS` покажет OCI-источник).

Commands you'll need (look them up — `helm package`, `helm registry login`, `helm push`, `helm install oci://...`).

Команды, которые понадобятся (найди их сам: `helm package`, `helm registry login`, `helm push`, `helm install oci://...`).

Paste into `HELM.md`:

Вставь в `HELM.md`:

- The `helm package` output
  * вывод `helm package`
- The `helm push` output **with the digest**
  * вывод `helm push` **с digest**
- The GHCR package URL (e.g. `https://github.com/users/<you>/packages/container/package/charts%2Flab10-app`)
  * URL пакета GHCR (например, `https://github.com/users/<you>/packages/container/package/charts%2Flab10-app`)
- The `helm install oci://…` output and `helm list` showing the OCI-sourced release
  * вывод `helm install oci://…` и `helm list` с релизом, установленным из OCI

---

## How to Submit

Как сдавать работу

Same flow as Lab 9: branch `lab10`, add `k8s/lab10-app/`, commit, push, open **two** PRs.

Тот же процесс, что в лабе 9: ветка `lab10`, добавь `k8s/lab10-app/`, закоммить, запушь, открой **два** PR.

```bash
git switch -c lab10
git add k8s/lab10-app/
git commit -m "feat(lab10): package web + echo as a helm 4 chart"
git push -u origin lab10
```

Пояснение: `git switch -c lab10` создаёт ветку и сразу переключается на неё, `git add` добавляет каталог чарта в индекс, `git commit` фиксирует изменения, `git push -u origin lab10` отправляет ветку в удалённый репозиторий и связывает её с источником.

PRs:

Пул-реквесты:

- `your-fork:lab10` → `course-repo:master` *(reviewed)*
  * `your-fork:lab10` → `course-repo:master` *(с ревью)*
- `your-fork:lab10` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab10` → `your-fork:master` *(сливается в твой собственный main по завершении)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — helm v4.1.x, public chart inspected, concepts explained
- [ ] Task 2 done — Chart.yaml, values.yaml, _helpers.tpl (4 templates), deployment + service + echo manifests, lint clean, REVISION 1 installed
- [ ] Task 3 done — values-prod.yaml + `helm upgrade -f base -f override` → REVISION 2 with replica count change
- [ ] Task 4 done — pre-install/pre-upgrade hook with the four annotations, hook fires + cleans up
- [ ] Task 5 done — HELM.md with all seven sections + evidence
- [ ] Bonus done — chart pushed to GHCR via OCI and installed from the registry
```

Пояснение: этот чек-лист вставляется в описание PR — отмечай пункты по мере выполнения. В нём перечислено всё, что должно быть готово к сдаче: задания 1–5, бонус, чистая проверка `helm lint` и установка на REVISION 1.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (1 pt)

Задание 1 (1 балл)

- ✅ `helm version` reports v4.1.x
  * `helm version` показывает v4.1.x
- ✅ A public OCI chart inspected via `helm show chart` / `helm show values`
  * Публичный OCI-чарт изучен через `helm show chart` / `helm show values`
- ✅ Chart / Release / Values explained in your own words
  * Chart / Release / Values объяснены своими словами

### Task 2 (3 pts)

Задание 2 (3 балла)

- ✅ `Chart.yaml` with the correct `apiVersion`, SemVer `version`, and `appVersion`
  * `Chart.yaml` с корректными `apiVersion`, SemVer-`version` и `appVersion`
- ✅ `values.yaml` with every Lab 9 hardcoded value exposed (replicas, image, ports, resources, probes, echo block)
  * `values.yaml`, где вынесено каждое захардкоженное значение из лабы 9 (реплики, образ, порты, ресурсы, пробы, блок echo)
- ✅ `_helpers.tpl` defines and uses **four** named templates: `name`, `fullname`, `labels`, `selectorLabels`
  * `_helpers.tpl` определяет и использует **четыре** именованных шаблона: `name`, `fullname`, `labels`, `selectorLabels`
- ✅ `selectorLabels` is a strict subset of `labels` (no `version` in it)
  * `selectorLabels` — строгое подмножество `labels` (без `version`)
- ✅ All four manifest templates render via `include`, `nindent`, `default`, `with`
  * Все четыре шаблона манифестов рендерятся через `include`, `nindent`, `default`, `with`
- ✅ Liveness + readiness probes kept and value-configurable
  * Пробы liveness и readiness сохранены и настраиваются через значения
- ✅ `helm lint` is clean and `helm install` reaches REVISION 1
  * `helm lint` проходит чисто, а `helm install` доходит до REVISION 1

### Task 3 (3 pts)

Задание 3 (3 балла)

- ✅ `values-prod.yaml` (and optionally dev/staging) contains **only deltas** from `values.yaml`
  * `values-prod.yaml` (и при желании dev/staging) содержит **только отличия** от `values.yaml`
- ✅ `helm upgrade ... -f values.yaml -f values-prod.yaml` produces REVISION 2
  * `helm upgrade ... -f values.yaml -f values-prod.yaml` даёт REVISION 2
- ✅ A real value (replica count) visibly changes between revisions
  * Реальное значение (число реплик) заметно меняется между ревизиями
- ✅ `helm rollback` returns the release to REVISION 1
  * `helm rollback` возвращает релиз к REVISION 1

### Task 4 (2 pts)

Задание 4 (2 балла)

- ✅ One Job hook with `helm.sh/hook: pre-install,pre-upgrade`
  * Один Job-хук с `helm.sh/hook: pre-install,pre-upgrade`
- ✅ `helm.sh/hook-weight` set (quoted string)
  * Задан `helm.sh/hook-weight` (строкой в кавычках)
- ✅ `helm.sh/hook-delete-policy` set so the Job is cleaned up
  * Задана `helm.sh/hook-delete-policy`, чтобы Job очищался
- ✅ Hook fires on both install and upgrade; logs captured
  * Хук срабатывает и при установке, и при обновлении; логи сохранены

### Task 5 (1 pt)

Задание 5 (1 балл)

- ✅ `HELM.md` covers all seven required sections with the captures pasted in
  * `HELM.md` покрывает все семь обязательных разделов с вставленными доказательствами

### Bonus (2 pts)

Бонус (2 балла)

- ✅ Chart packaged to `.tgz`
  * Чарт упакован в `.tgz`
- ✅ Pushed to `oci://ghcr.io/<username>/charts`
  * Запушен в `oci://ghcr.io/<username>/charts`
- ✅ Installed *from the registry* (not from the local directory)
  * Установлен *из реестра* (а не из локального каталога)
- ✅ GHCR URL + install-from-OCI output in `HELM.md`
  * URL GHCR и вывод установки из OCI есть в `HELM.md`

---

## Rubric

Рубрика оценивания

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Fundamentals | **1** | v4.1.x, public chart inspected, concepts explained |
| **Task 2** — Chart build | **3** | Clean templating, four helpers, lint clean, install reaches REVISION 1 |
| **Task 3** — Multi-env | **3** | `-f a -f b` layering produces REVISION 2 with a visible value change; rollback works |
| **Task 4** — Hooks | **2** | Pre-install/pre-upgrade hook with weight + delete policy; fires and cleans up |
| **Task 5** — Documentation | **1** | Complete `HELM.md` with evidence |
| **Bonus** — OCI push | **2** | Chart packaged, pushed to GHCR, installed *from the registry* |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Основы | **1** | v4.1.x, публичный чарт изучен, концепции объяснены |
| **Задание 2** — Сборка чарта | **3** | Чистое шаблонизирование, четыре хелпера, чистый lint, установка до REVISION 1 |
| **Задание 3** — Несколько окружений | **3** | Наложение `-f a -f b` даёт REVISION 2 с видимым изменением значения; откат работает |
| **Задание 4** — Хуки | **2** | Хук pre-install/pre-upgrade с весом и политикой удаления; срабатывает и очищается |
| **Задание 5** — Документация | **1** | Полный `HELM.md` с доказательствами |
| **Бонус** — OCI-пуш | **2** | Чарт упакован, запушен в GHCR, установлен *из реестра* |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading:**

**Оценивание:**

- **10/10:** Clean templating from scratch, multi-env upgrade visible across revisions, hook fires + cleans up, thorough docs
  * **10/10:** чистое шаблонизирование с нуля, видимое обновление нескольких окружений между ревизиями, хук срабатывает и очищается, подробная документация
- **8–9/10:** Chart installs and upgrades, minor gaps in helpers or hook annotations
  * **8–9/10:** чарт устанавливается и обновляется, небольшие пробелы в хелперах или аннотациях хука
- **6–7/10:** Basic chart installs but multi-env upgrade or hook polish missing
  * **6–7/10:** базовый чарт устанавливается, но обновление нескольких окружений или доводка хука отсутствуют
- **<6/10:** `helm create` boilerplate submitted, hardcoded values, no hook, or chart doesn't lint
  * **<6/10:** сдан шаблонный код `helm create`, захардкоженные значения, нет хука или чарт не проходит lint

---

## Resources

Ресурсы

<details>
<summary>📚 Documentation</summary>

📚 Документация

- [Helm Documentation](https://helm.sh/docs/) — Helm 4
  * [Документация Helm](https://helm.sh/docs/) — Helm 4
- [Chart Template Guide](https://helm.sh/docs/chart_template_guide/)
  * [Руководство по шаблонам чартов](https://helm.sh/docs/chart_template_guide/)
- [Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
  * [Лучшие практики для чартов](https://helm.sh/docs/chart_best_practices/)
- [Chart Hooks](https://helm.sh/docs/topics/charts_hooks/)
  * [Хуки чартов](https://helm.sh/docs/topics/charts_hooks/)
- [Use OCI-based registries](https://helm.sh/docs/topics/registries/)
  * [Использование реестров на основе OCI](https://helm.sh/docs/topics/registries/)
- [Sprig function reference](https://masterminds.github.io/sprig/) — the function library you'll consult weekly
  * [Справочник функций Sprig](https://masterminds.github.io/sprig/) — библиотека функций, к которой ты будешь обращаться каждую неделю
- [Built-in Objects](https://helm.sh/docs/chart_template_guide/builtin_objects/) — `.Release`, `.Chart`, `.Values`, `.Files`, `.Capabilities`
  * [Встроенные объекты](https://helm.sh/docs/chart_template_guide/builtin_objects/) — `.Release`, `.Chart`, `.Values`, `.Files`, `.Capabilities`

</details>

<details>
<summary>🛠️ Tools & Registries</summary>

🛠️ Инструменты и реестры

- [Artifact Hub](https://artifacthub.io/) — public chart search (15,000+ charts)
  * [Artifact Hub](https://artifacthub.io/) — поиск публичных чартов (более 15 000 чартов)
- [helm-docs](https://github.com/norwoodj/helm-docs) — auto-generate README from values
  * [helm-docs](https://github.com/norwoodj/helm-docs) — автоматическая генерация README из values
- [chart-testing](https://github.com/helm/chart-testing) — lint & install charts in CI
  * [chart-testing](https://github.com/helm/chart-testing) — lint и установка чартов в CI
- [GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) — OCI registry for both images and charts
  * [GHCR](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) — OCI-реестр и для образов, и для чартов

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Типичные ошибки (из реальных прогонов)

- **`helm create` overwrites your work.** Running it inside an existing chart directory clobbers your hand-written `Chart.yaml`, `values.yaml`, and helpers without asking. Don't run it. Write the files yourself — this lab grades that skill.
  * **`helm create` перезаписывает твою работу.** Запуск в существующем каталоге чарта без вопросов затирает твои вручную написанные `Chart.yaml`, `values.yaml` и хелперы. Не запускай его. Пиши файлы сам — именно этот навык и оценивается в лабе.
- **Fullname helper missing the collapse branch.** A naive `printf "%s-%s" .Release.Name .Chart.Name` produces `lab10-app-lab10-app` when release name and chart name match (which they do — release is `lab10-app`). Then `-web` appends to give `lab10-app-lab10-app-web`. Lab 11 references `deploy/lab10-app-web`, so this breaks the next lab silently. The canonical `if contains $name .Release.Name` branch (cribbed from `helm create`) is what makes the collapse work. Test with `helm template lab10-app k8s/lab10-app | grep '^  name:'` BEFORE installing.
  * **В хелпере fullname нет ветки схлопывания.** Наивный `printf "%s-%s" .Release.Name .Chart.Name` даёт `lab10-app-lab10-app`, когда имя релиза совпадает с именем чарта (а они совпадают — релиз называется `lab10-app`). Потом добавляется `-web`, и получается `lab10-app-lab10-app-web`. Лаба 11 обращается к `deploy/lab10-app-web`, так что следующая лаба молча сломается. Каноническая ветка `if contains $name .Release.Name` (подсмотренная в `helm create`) как раз и обеспечивает схлопывание. Проверь это через `helm template lab10-app k8s/lab10-app | grep '^  name:'` ДО установки.
- **Replacing the whole `helm create` `values.yaml` breaks `helm lint`.** The generated boilerplate ships a `templates/serviceaccount.yaml` that references `.Values.serviceAccount.create`. The moment you blow away that block in `values.yaml`, `helm lint` errors out because the referenced key vanished. Either edit the defaults in place **or** (recommended) skip `helm create` entirely. *(This is the exact self-inflicted snag from the lab's own reference run.)*
  * **Полная замена `values.yaml` из `helm create` ломает `helm lint`.** В сгенерированном шаблоне есть `templates/serviceaccount.yaml`, ссылающийся на `.Values.serviceAccount.create`. Как только ты убираешь этот блок из `values.yaml`, `helm lint` падает, потому что исчез упомянутый ключ. Либо правь значения на месте, **либо** (рекомендуется) вообще не запускай `helm create`. *(Это ровно та проблема, которую авторы лабы создали себе сами на эталонном прогоне.)*
- **`indent` vs `nindent`.** `nindent N` = newline + N spaces; `indent N` = N spaces only. When a `{{- ... -}}` action has already eaten the previous newline, `indent` produces invalid YAML that `helm lint` will catch and `helm install` will refuse. Use `nindent` unless you're sure.
  * **`indent` против `nindent`.** `nindent N` = перевод строки + N пробелов; `indent N` = только N пробелов. Когда действие `{{- ... -}}` уже «съело» предыдущий перевод строки, `indent` даёт невалидный YAML: `helm lint` его поймает, а `helm install` откажется работать. Используй `nindent`, если не уверен.
- **Mutable selector labels break rolling updates.** Once a `Deployment` exists, K8s rejects changes to `spec.selector.matchLabels`. If `version` is in your selector labels, every `appVersion` bump triggers a chart upgrade that K8s refuses — you'll have to `kubectl delete deploy ...` and lose zero-downtime. Keep `selectorLabels` to `name` + `instance` only.
  * **Изменяемые селекторные метки ломают rolling-обновления.** Как только `Deployment` создан, K8s отклоняет изменения `spec.selector.matchLabels`. Если `version` попала в селекторные метки, каждое повышение `appVersion` будет пытаться обновить чарт, а K8s откажет — придётся делать `kubectl delete deploy ...` и потерять zero-downtime. В `selectorLabels` оставляй только `name` + `instance`.
- **`hook-weight` must be a quoted string.** `"helm.sh/hook-weight": -5` (unquoted) silently parses as a number and Helm ignores it; you'll get non-deterministic hook ordering. Always quote: `"helm.sh/hook-weight": "-5"`.
  * **`hook-weight` должен быть строкой в кавычках.** `"helm.sh/hook-weight": -5` (без кавычек) молча разберётся как число, и Helm его проигнорирует; порядок хуков станет недетерминированным. Всегда бери в кавычки: `"helm.sh/hook-weight": "-5"`.
- **Hook resources outlive `helm uninstall`.** Hook Jobs are *not* part of the release. Without `hook-delete-policy`, failed Jobs pile up forever and `helm uninstall` won't remove them. Always set `before-hook-creation,hook-succeeded`.
  * **Ресурсы хуков переживают `helm uninstall`.** Job-хуки *не* входят в состав релиза. Без `hook-delete-policy` неудачные Job копятся бесконечно, и `helm uninstall` их не удалит. Всегда задавай `before-hook-creation,hook-succeeded`.
- **Forgetting `helm dependency update` after editing `Chart.yaml`.** If you add a `dependencies:` entry (not required in this lab, but easy to add for the bonus), the subchart isn't downloaded into `charts/` until you run `helm dependency update`. `helm install` then fails with a missing-dependency error that's confusingly worded.
  * **Забытый `helm dependency update` после правки `Chart.yaml`.** Если добавишь запись `dependencies:` (в этой лабе не требуется, но легко добавить ради бонуса), сабчарт не окажется в `charts/`, пока ты не выполнишь `helm dependency update`. Тогда `helm install` упадёт с ошибкой о недостающей зависимости, сформулированной путано.
- **OCI auth gotcha.** `helm registry login ghcr.io` uses Docker config, not a separate Helm credential store. If a stale Docker login exists for `ghcr.io`, you may need `docker logout ghcr.io` first. Always pipe the PAT via `--password-stdin`; never put it on the command line (it lands in your shell history).
  * **Ловушка авторизации OCI.** `helm registry login ghcr.io` использует конфиг Docker, а не отдельное хранилище учётных данных Helm. Если в Docker остался устаревший логин для `ghcr.io`, сначала может понадобиться `docker logout ghcr.io`. Всегда передавай PAT через `--password-stdin`; никогда не указывай его в командной строке (он попадёт в историю shell).
- **`apiVersion: v2` vs `v1` in `Chart.yaml`.** `v2` is the chart-format version introduced by Helm 3 and still used by Helm 4 — **not** the Helm binary version. `v1` is Helm 2 only. Helm 4 against a `v1` chart fails immediately; Helm 3 against a `v2` chart works. Always write `apiVersion: v2`.
  * **`apiVersion: v2` против `v1` в `Chart.yaml`.** `v2` — версия формата чарта, введённая в Helm 3 и по-прежнему используемая в Helm 4, — **не** версия бинарника Helm. `v1` существует только в Helm 2. Helm 4 с чартом `v1` падает сразу; Helm 3 с чартом `v2` работает. Всегда пиши `apiVersion: v2`.
- **Empty `image.tag` in `values.yaml`.** Setting `tag: ""` lets `{{ .Values.image.tag | default .Chart.AppVersion }}` fall back. Setting `tag: latest` defeats reproducibility (kills rollbacks and breaks ArgoCD diff in Lab 13). Leave it empty in defaults; pin a SemVer in `values-prod.yaml`.
  * **Пустой `image.tag` в `values.yaml`.** `tag: ""` позволяет `{{ .Values.image.tag | default .Chart.AppVersion }}` подхватить значение из `appVersion`. `tag: latest` убивает воспроизводимость (ломает откаты и ломает diff в ArgoCD в лабе 13). Оставляй его пустым в значениях по умолчанию, а в `values-prod.yaml` закрепляй SemVer.

</details>

<details>
<summary>📖 Learning Resources</summary>

📖 Учебные материалы

- [Quickstart Guide](https://helm.sh/docs/intro/quickstart/)
  * [Быстрый старт](https://helm.sh/docs/intro/quickstart/)
- [Using Helm](https://helm.sh/docs/intro/using_helm/)
  * [Использование Helm](https://helm.sh/docs/intro/using_helm/)
- [Three Big Concepts](https://helm.sh/docs/intro/using_helm/#three-big-concepts)
  * [Три больших концепции](https://helm.sh/docs/intro/using_helm/#three-big-concepts)
- *Learning Helm* (2e, 2024) — Butcher, Farina, Dolitsky (O'Reilly)
  * *Learning Helm* (2-е изд., 2024) — Батчер, Фарина, Долицки (O'Reilly)

</details>

---

## Looking Ahead

Что дальше

| Lab | What it adds to this chart |
|---:|---|
| 11 | Secrets management with OpenBao — because `values.yaml` is git-tracked and your DB password isn't |
| 12 | ConfigMaps + PVCs added as new templates in this same chart |
| 13 | ArgoCD deploys this chart via GitOps; `helm template` becomes the rendering step |
| 14 | Argo Rollouts replaces the Deployment template with a Rollout for canary delivery |
| 16 | kube-prometheus-stack (a Helm chart itself) scrapes your app via a ServiceMonitor you add to this chart |

| Лаба | Что добавляет к этому чарту |
|---:|---|
| 11 | Управление секретами с OpenBao — потому что `values.yaml` хранится в git, а пароль от БД там быть не должен |
| 12 | ConfigMaps + PVC как новые шаблоны в этом же чарте |
| 13 | ArgoCD разворачивает этот чарт через GitOps; `helm template` становится шагом рендеринга |
| 14 | Argo Rollouts заменяет шаблон Deployment на Rollout для канареечной доставки |
| 16 | kube-prometheus-stack (сам по себе Helm-чарт) собирает метрики твоего приложения через ServiceMonitor, который ты добавишь в этот чарт |

---

**Good luck!** ⛵

**Удачи!** ⛵

> **Remember:** Template everything, hardcode nothing (except sensible defaults in `values.yaml`). One chart, N value files. Never comment out a health check — make it a value instead. And **never** run `helm create` inside this lab.
>
> **Помни:** шаблонизируй всё, не хардкодь ничего (кроме разумных значений по умолчанию в `values.yaml`). Один чарт, N файлов значений. Никогда не отключай проверку работоспособности, комментируя её, — лучше вынеси её в значение. И **никогда** не запускай `helm create` внутри этой лабы.
