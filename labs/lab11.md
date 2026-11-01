# Lab 11 — Kubernetes Secrets & OpenBao

Лаба 11 — Kubernetes Secrets и OpenBao

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Secret%20Management-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-OpenBao%202.5%20%7C%20K8s%20Secrets-informational)

> **Goal:** Stop putting passwords in Git. Prove to yourself that a Kubernetes `Secret` is **not** encryption, then centralize secrets in **OpenBao 2.5.0** with a read-only policy enforcing least privilege.
> **Deliverable:** A PR from `lab11` adding `k8s/secrets/` (the policy + sample Secret), `k8s/lab10-app/templates/secret.yaml` to your Lab 10 chart, and `docs/LAB11.md` with **real** evidence (your decoded values, your `bao kv get`, your denied write).
>
> **Цель:** перестать хранить пароли в Git. Убедиться на практике, что Kubernetes `Secret` — это **не** шифрование, а затем централизовать секреты в **OpenBao 2.5.0** с политикой только на чтение, реализующей принцип наименьших привилегий.
> **Что сдать:** PR из ветки `lab11`, который добавляет `k8s/secrets/` (политику и пример Secret), `k8s/lab10-app/templates/secret.yaml` в чарт из лабы 10 и `docs/LAB11.md` с **реальными** доказательствами (твои декодированные значения, твой вывод `bao kv get`, твой отказ в записи).

---

## Overview

Обзор

In this lab you will practice:

В этой лабе ты отработаешь:

- Creating a `kind: Secret` and **decoding its base64 yourself** to drive home that base64 ≠ encryption
  * Создание `kind: Secret` и **самостоятельное декодирование base64**, чтобы окончательно усвоить: base64 ≠ шифрование
- Bringing up **OpenBao 2.5.0** in dev mode and learning the env-var contract (`BAO_ADDR`, `BAO_TOKEN`) that every later production setup will reuse
  * Запуск **OpenBao 2.5.0** в dev-режиме и освоение контракта переменных окружения (`BAO_ADDR`, `BAO_TOKEN`), который потом будет переиспользовать любая продакшен-конфигурация
- Writing a **read-only HCL policy** and a token bound to it, then proving the policy by trying — and failing — to write
  * Написание **HCL-политики только на чтение** и токена, привязанного к ней, а затем проверка политики попыткой записи — которая должна провалиться
- Templating a `Secret` into your Lab 10 chart **without** committing the real value (the value comes from `--set`/`-f` at install time)
  * Шаблонизация `Secret` в чарте из лабы 10 **без** коммита реального значения (значение передаётся через `--set`/`-f` при установке)
- Picking one of two production patterns for injection: **OpenBao Agent Injector** OR **External Secrets Operator (ESO)**
  * Выбор одного из двух продакшен-паттернов доставки секретов: **OpenBao Agent Injector** ИЛИ **External Secrets Operator (ESO)**

> ⚠️ **Scope:** dev-mode OpenBao only (in-memory, single key, auto-unsealed). Production OpenBao (Raft storage, auto-unseal via cloud KMS, audit device, TLS) is out of scope — but the policies and roles you write here are the same shape.
>
> ⚠️ **Границы лабы:** только OpenBao в dev-режиме (в памяти, один ключ, автоматический unseal). Продакшен-OpenBao (хранилище Raft, auto-unseal через облачный KMS, audit device, TLS) вне рамок — но политики и роли, которые ты здесь напишешь, имеют ту же форму.

> 💡 **The five incidents the lecture opened with — Code Spaces 2014, Uber 2016, Toyota 2022, Dropbox 2022, tj-actions 2025 — every single one was a credential stored in the wrong place.** Not a zero-day. Not a sophisticated 0-click. Just a secret in code, a `.git/` directory served by a web server, a phished GitHub account, a compromised GitHub Action dumping `env` to logs. This lab teaches the boring stuff that would have prevented all five.
>
> 💡 **Пять инцидентов, с которых началась лекция — Code Spaces 2014, Uber 2016, Toyota 2022, Dropbox 2022, tj-actions 2025 — и в каждом из них учётные данные хранились не там, где нужно.** Не zero-day. Не изощрённая атака в 0 кликов. Просто секрет в коде, каталог `.git/`, отданный веб-сервером, угнанный через фишинг GitHub-аккаунт, скомпрометированный GitHub Action, сливающий `env` в логи. Эта лаба учит скучным вещам, которые предотвратили бы все пять случаев.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно остаться от предыдущих лаб:**

- Lab 9: a k3d 1.36 cluster (`k3d cluster create devops`) with your `web` + `echo` services running
  * Лаба 9: кластер k3d 1.36 (`k3d cluster create devops`) с работающими сервисами `web` и `echo`
- Lab 10: a Helm chart at `k8s/lab10-app/` with `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`
  * Лаба 10: Helm-чарт в `k8s/lab10-app/` с `Chart.yaml`, `values.yaml`, `templates/`, `_helpers.tpl`

**This lab adds:**

**Эта лаба добавляет:**

- `k8s/secrets/app-credentials.yaml` — your hand-written `Secret` manifest (the base64 demo)
  * `k8s/secrets/app-credentials.yaml` — написанный вручную манифест `Secret` (демонстрация base64)
- `k8s/lab10-app/templates/secret.yaml` — a templated `Secret` in your chart
  * `k8s/lab10-app/templates/secret.yaml` — шаблонизированный `Secret` в твоём чарте
- `k8s/secrets/lab11-read.hcl` — the OpenBao read-only policy
  * `k8s/secrets/lab11-read.hcl` — политика OpenBao только на чтение
- `docs/LAB11.md` — your submission report with the captured CLI evidence
  * `docs/LAB11.md` — отчёт по лабе с зафиксированными доказательствами из CLI
- *(bonus)* either `k8s/secrets/injector.yaml` (Agent Injector annotations) OR `k8s/secrets/eso.yaml` (SecretStore + ExternalSecret CRDs)
  * *(бонус)* либо `k8s/secrets/injector.yaml` (аннотации Agent Injector), либо `k8s/secrets/eso.yaml` (CRD SecretStore + ExternalSecret)

---

## Setup

Подготовка

You need (verify before starting):

Тебе понадобится (проверь перед началом):

- `kubectl version --client` → 1.36.x (from Lab 9)
  * `kubectl version --client` → 1.36.x (из лабы 9)
- `helm version` → 4.1.x (from Lab 10)
  * `helm version` → 4.1.x (из лабы 10)
- `k3d cluster list` shows the `devops` cluster from Lab 9 Running
  * `k3d cluster list` показывает кластер `devops` из лабы 9 в состоянии Running
- **OpenBao 2.5.0** installed locally. Install via the brew tap (`brew install openbao/tap/bao`) or by downloading the `v2.5.0` Linux binary tarball from `github.com/openbao/openbao/releases` and unpacking it onto `PATH`.
  * **OpenBao 2.5.0**, установленный локально. Ставится через brew tap (`brew install openbao/tap/bao`) или скачиванием архива с Linux-бинарником `v2.5.0` с `github.com/openbao/openbao/releases` с распаковкой в `PATH`.
- `bao version` must print `OpenBao v2.5.0`
  * `bao version` должен вывести `OpenBao v2.5.0`

Create the namespace + ServiceAccount that later tasks reuse:

Создай namespace и ServiceAccount, которые понадобятся в следующих задачах:

```bash
kubectl create namespace lab11
kubectl create serviceaccount lab11-sa -n lab11
```

Пояснение: создаётся отдельный namespace `lab11` и ServiceAccount `lab11-sa`, который позже будет аутентифицироваться в OpenBao через Kubernetes auth.

> **`bao` is a drop-in for `vault`.** OpenBao kept the wire protocol, CLI subcommands, and API endpoints. If you have old `vault kv put ...` muscle memory it still works against an OpenBao server. We use `bao` in this lab because the binary you installed is OpenBao, not HashiCorp Vault.
>
> **`bao` — это drop-in замена `vault`.** OpenBao сохранил протокол, подкоманды CLI и эндпоинты API. Если у тебя есть мышечная память от старых `vault kv put ...`, она сработает и с сервером OpenBao. В этой лабе мы используем `bao`, потому что установленный бинарник — это OpenBao, а не HashiCorp Vault.

> 📜 **One-paragraph BSL callout.** In August 2023 HashiCorp re-licensed Vault under the Business Source License 1.1 — source-available, but with a non-compete that forbids commercial managed-service offerings competing with HashiCorp. The Linux Foundation forked Vault 1.14 as **OpenBao** under MPL-2.0 (true open source). OpenBao 2.0 went GA in March 2024; 2.5.0 (Feb 4 2026) added free Namespaces and horizontal read scalability — features that used to be a HashiCorp Enterprise paywall. Everything you write in this lab is portable to either server; we standardize on OpenBao so the manifests survive any future re-licensing.
>
> 📜 **Одна короткая ремарка про BSL.** В августе 2023 года HashiCorp перелицензировал Vault под Business Source License 1.1 — исходники открыты, но с запретом на конкурентные коммерческие managed-сервисы, конкурирующие с HashiCorp. Linux Foundation форкнул Vault 1.14 как **OpenBao** под MPL-2.0 (честный open source). OpenBao 2.0 вышел в GA в марте 2024-го; в 2.5.0 (4 февраля 2026) появились бесплатные Namespaces и горизонтальное масштабирование чтения — возможности, которые раньше были за платным доступом в HashiCorp Enterprise. Всё, что ты напишешь в этой лабе, переносимо на любой из серверов; мы стандартизируемся на OpenBao, чтобы манифесты пережили любую будущую перелицензировку.

---

## Task 1 — Kubernetes Secrets & The Base64 Trap (2 pts)

Задача 1 — Kubernetes Secrets и ловушка base64 (2 балла)

### 1.1 — Hand-write a `kind: Secret` manifest

1.1 — Написать вручную манифест `kind: Secret`

`YOUR TASK`: create `k8s/secrets/app-credentials.yaml` containing a `kind: Secret` for a fictional database with username `app` and password of your choosing. Use the **declarative** form (a YAML file you `kubectl apply`), not `kubectl create secret`.

ТВОЯ ЗАДАЧА: создай `k8s/secrets/app-credentials.yaml` с `kind: Secret` для вымышленной базы данных с именем пользователя `app` и паролем на твой выбор. Используй **декларативную** форму (YAML-файл, который применяешь через `kubectl apply`), а не `kubectl create secret`.

You decide:

Что решаешь ты:

- The `type:` value — pick from `Opaque`, `kubernetes.io/dockerconfigjson`, or `kubernetes.io/tls`. The wrong choice fails validation, so the choice matters. In `docs/LAB11.md`, justify why your choice fits a generic username+password.
  * Значение `type:` — выбери из `Opaque`, `kubernetes.io/dockerconfigjson` или `kubernetes.io/tls`. Неверный выбор не пройдёт валидацию, так что выбор важен. В `docs/LAB11.md` обоснуй, почему твой выбор подходит для обычной пары логин+пароль.
- The two `data:` keys — name them whatever a real app would consume (think env-var style).
  * Два ключа в `data:` — назови их так, как их ожидало бы реальное приложение (в стиле имён переменных окружения).
- The base64-encoded values — compute them yourself with `base64`; do **not** use the `stringData:` shortcut for this task. The point is to feel the encoding step by hand.
  * Значения в base64 — вычисли их сам через `base64`; **не** используй в этой задаче сокращение через `stringData:`. Смысл именно в том, чтобы прочувствовать шаг кодирования руками.

Skeleton (fill the YOUR-TASK markers):

Скелет (заполни маркеры YOUR-TASK):

```yaml
# k8s/secrets/app-credentials.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  namespace: lab11
type: YOUR-TASK                 # which built-in type fits username+password?
data:
  YOUR-TASK: YOUR-TASK          # echo -n 'app' | base64
  YOUR-TASK: YOUR-TASK          # echo -n '<your password>' | base64
```

Пояснение: это заготовка манифеста — заполни `type` и два ключа `data`, закодировав значения в base64.

> ⚠️ **`echo` vs `echo -n`** — `echo "foo" | base64` includes the trailing newline (`Zm9vCg==`); `echo -n "foo" | base64` does not (`Zm9v`). The newline silently breaks password comparisons in the consuming app and is the most common base64 bug in dry-runs. Use `-n`.
>
> ⚠️ **`echo` против `echo -n`** — `echo "foo" | base64` включает перевод строки в конце (`Zm9vCg==`); `echo -n "foo" | base64` — нет (`Zm9v`). Этот перевод строки молча ломает сравнение паролей в приложении-потребителе и является самой частой ошибкой base64 на прогонах. Используй `-n`.

Apply it, then read it back:

Примени его, затем прочитай обратно:

```bash
kubectl apply -f k8s/secrets/app-credentials.yaml
kubectl get secret app-credentials -n lab11 -o yaml
```

Пояснение: `kubectl apply` создаёт Secret из декларативного файла, а `kubectl get ... -o yaml` показывает его состояние так, как оно хранится в кластере.

### 1.2 — Decode it yourself and prove the point

1.2 — Декодируй сам и докажи мысль

`YOUR TASK`: run a one-liner that fetches **your own** Secret from the cluster and pipes the password field through `base64 -d`, recovering the plaintext. Capture the output verbatim into `docs/LAB11.md` under a heading **"The base64 'aha' moment"**.

ТВОЯ ЗАДАЧА: выполни однострочник, который достаёт **твой собственный** Secret из кластера, прогоняет поле пароля через `base64 -d` и возвращает открытый текст. Скопируй вывод дословно в `docs/LAB11.md` под заголовком **«Момент озарения с base64»**.

Hint (the shape of the command — fill in the key name you chose):

Подсказка (форма команды — подставь имя ключа, которое ты выбрал):

```bash
kubectl get secret app-credentials -n lab11 \
  -o jsonpath='{.data.YOUR-TASK}' | base64 -d ; echo
```

Пояснение: `jsonpath` достаёт нужное поле в base64, `base64 -d` декодирует его обратно в открытый текст, а `echo` добавляет перевод строки для читаемости.

In **2–3 sentences** in your `docs/LAB11.md`, answer: *why does the K8s API even use base64 here if it's not for security?* (Hint: `Secret.data` values must be valid JSON/YAML strings; binary like a TLS key wouldn't survive transport.)

В **2–3 предложениях** в `docs/LAB11.md` ответь: *зачем Kubernetes API вообще использует здесь base64, если это не для безопасности?* (Подсказка: значения `Secret.data` должны быть корректными строками JSON/YAML; бинарные данные вроде ключа TLS не пережили бы передачу.)

### 1.3 — etcd encryption-at-rest, in your own words

1.3 — Шифрование etcd при хранении: своими словами

Write a paragraph in `docs/LAB11.md` answering all three:

Напиши абзац в `docs/LAB11.md`, отвечающий на все три вопроса:

- Are Kubernetes Secrets encrypted at rest by default? (Hint: look up `EncryptionConfiguration`.)
  * Шифруются ли Kubernetes Secrets при хранении по умолчанию? (Подсказка: посмотри `EncryptionConfiguration`.)
- What does a KMS provider in that config do that the `aescbc` provider doesn't?
  * Что делает KMS-провайдер в этой конфигурации, чего не делает провайдер `aescbc`?
- Why is enabling `EncryptionConfiguration` necessary but **not sufficient** for production — what does it not protect you from? (Hint: anyone with `system:masters` RBAC.)
  * Почему включение `EncryptionConfiguration` необходимо, но **недостаточно** для продакшена — от чего оно не защищает? (Подсказка: кто угодно с правами `system:masters` в RBAC.)

### 1.4 — Proof of work

1.4 — Доказательство работы

Paste into `docs/LAB11.md`:

Вставь в `docs/LAB11.md`:

- The contents of `k8s/secrets/app-credentials.yaml` (your real file, your real base64 strings)
  * Содержимое `k8s/secrets/app-credentials.yaml` (твой настоящий файл, твои настоящие строки base64)
- The output of `kubectl get secret app-credentials -n lab11 -o yaml | grep -A2 ^data:`
  * Вывод `kubectl get secret app-credentials -n lab11 -o yaml | grep -A2 ^data:`
- The base64-decode one-liner **and its plaintext output** showing you recovered your own password
  * Однострочник декодирования base64 **и его открытый вывод**, показывающий, что ты восстановил свой пароль
- Your 2–3 sentence answer to *why base64?* and the etcd-at-rest paragraph from 1.3
  * Твой ответ из 2–3 предложений на вопрос *зачем base64?* и абзац про etcd-at-rest из 1.3

---

## Task 2 — Helm-Managed Secrets (3 pts)

Задача 2 — Секреты под управлением Helm (3 балла)

### 2.1 — Templated Secret in the Lab 10 chart

2.1 — Шаблонизированный Secret в чарте из лабы 10

`YOUR TASK`: add `templates/secret.yaml` to your Lab 10 chart (`k8s/lab10-app/templates/secret.yaml`). It must render a `kind: Secret` named after the chart's `fullname` helper, carry the standard chart labels, and consume **two** values from `.Values.secret` — `dbUsername` and `dbPassword`.

ТВОЯ ЗАДАЧА: добавь `templates/secret.yaml` в чарт из лабы 10 (`k8s/lab10-app/templates/secret.yaml`). Он должен рендерить `kind: Secret` с именем из хелпера `fullname` чарта, нести стандартные лейблы чарта и использовать **два** значения из `.Values.secret` — `dbUsername` и `dbPassword`.

Critical rules:

Критические правила:

- Use `stringData:` (so Helm doesn't double-encode); the K8s API base64-encodes on its side.
  * Используй `stringData:` (чтобы Helm не закодировал дважды); K8s API сам закодирует в base64 на своей стороне.
- In `values.yaml`, set the two fields to **placeholder strings** (`"PLACEHOLDER_USER"`, `"PLACEHOLDER_PASS"`). The real values come from `--set` or a `-f override.yaml` at install time — *never* committed.
  * В `values.yaml` задай этим полям **строки-заглушки** (`"PLACEHOLDER_USER"`, `"PLACEHOLDER_PASS"`). Реальные значения приходят из `--set` или `-f override.yaml` при установке и *никогда* не коммитятся.

Skeleton (fill the YOUR-TASK markers):

Скелет (заполни маркеры YOUR-TASK):

```yaml
# k8s/lab10-app/templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: YOUR-TASK              # use the fullname helper from your _helpers.tpl
  labels:
    YOUR-TASK                  # use the labels helper from your _helpers.tpl
type: YOUR-TASK                # same choice as Task 1.1
stringData:
  DB_USERNAME: YOUR-TASK       # quote a value pulled from .Values.secret.*
  DB_PASSWORD: YOUR-TASK
```

Пояснение: имя и лейблы бери из хелперов `_helpers.tpl`, а `stringData` позволяет указать значения в открытом виде — API закодирует их сам.

And in `values.yaml`:

А в `values.yaml`:

```yaml
secret:
  dbUsername: "PLACEHOLDER_USER"
  dbPassword: "PLACEHOLDER_PASS"   # override at install time, NEVER commit real values
```

Пояснение: в репозитории лежат только заглушки; настоящий пароль подставляется при установке и в коммит не попадает.

### 2.2 — Consume it in the Deployment

2.2 — Используй Secret в Deployment

`YOUR TASK`: edit your chart's existing Deployment template (`templates/deployment.yaml`) so the `web` container reads both keys as environment variables. Use the `envFrom` + `secretRef` pattern (cleaner than per-key `secretKeyRef` when the whole Secret is for one consumer).

ТВОЯ ЗАДАЧА: отредактируй существующий шаблон Deployment своего чарта (`templates/deployment.yaml`), чтобы контейнер `web` читал оба ключа как переменные окружения. Используй паттерн `envFrom` + `secretRef` (он чище, чем `secretKeyRef` для каждого ключа, когда весь Secret предназначен одному потребителю).

Skeleton (fill the YOUR-TASK marker):

Скелет (заполни маркер YOUR-TASK):

```yaml
containers:
  - name: web
    image: # ... your Lab 10 image ...
    envFrom:
      - secretRef:
          name: YOUR-TASK      # same helper-rendered name as in Task 2.1
```

Пояснение: `envFrom.secretRef` подставит все ключи Secret как переменные окружения контейнера `web`.

### 2.3 — Install with the real value out-of-band

2.3 — Установка с реальным значением вне репозитория

`YOUR TASK`: install (or upgrade) the chart, supplying the real password via `--set` **or** a `-f local-secrets.yaml` that is `.gitignore`d. Then verify in the running pod.

ТВОЯ ЗАДАЧА: установи (или обнови) чарт, передав реальный пароль через `--set` **или** через `-f local-secrets.yaml`, который добавлен в `.gitignore`. Затем проверь результат в запущенном поде.

> ⚠️ **The `--set` CI-logs trap.** `helm upgrade --install ... --set secret.dbPassword='hunter2'` is fine on your laptop. In CI it ends up in the workflow log unless the variable is wrapped as a secret (`${{ secrets.DB_PASSWORD }}` in GHA) **and** you `helm upgrade ... --set secret.dbPassword="$DB_PASSWORD"` with no echo. Document in `docs/LAB11.md` which pattern you'd use in a real GitHub Action.
>
> ⚠️ **Ловушка `--set` в логах CI.** `helm upgrade --install ... --set secret.dbPassword='hunter2'` — это нормально на твоём ноутбуке. В CI это попадёт в лог workflow, если только переменная не оформлена как секрет (`${{ secrets.DB_PASSWORD }}` в GHA) **и** ты не вызываешь `helm upgrade ... --set secret.dbPassword="$DB_PASSWORD"` без эха. Опиши в `docs/LAB11.md`, какой паттерн использовал бы в реальном GitHub Action.

> 🔮 **Lab 13 preview — `--set` does NOT survive ArgoCD.** Next week ArgoCD will sync this same chart from Git, and it only honors `helm.valueFiles` / `helm.valuesObject` on the `Application` CR — there is **no `--set` knob** on a GitOps reconcile. That's why the bonus task installs **OpenBao Agent Injector** or **ESO**: they deliver the secret out-of-band so the chart in Git can ship with placeholders forever. If you skip the bonus, the lab 13 workaround is `helm.valuesObject` referencing an externally-managed Secret — never a real value in Git.
>
> 🔮 **Анонс лабы 13 — `--set` НЕ переживёт ArgoCD.** На следующей неделе ArgoCD будет синхронизировать этот же чарт из Git, а он учитывает только `helm.valueFiles` / `helm.valuesObject` в CR `Application` — **никакого переключателя `--set`** в GitOps-синхронизации нет. Именно поэтому в бонусной задаче ставится **OpenBao Agent Injector** или **ESO**: они доставляют секрет вне репозитория, так что чарт в Git может навсегда оставаться с заглушками. Если пропустишь бонус, обходной путь для лабы 13 — `helm.valuesObject` со ссылкой на управляемый извне Secret — но никогда не реальное значение в Git.

Verification commands (illustrative — your output will differ):

Команды для проверки (иллюстративные — твой вывод будет отличаться):

```bash
helm upgrade --install lab10-app k8s/lab10-app -n lab11 \
  --set secret.dbPassword='YOUR-PASSWORD'         # YOUR-TASK: pick a value

# Find the actual web Deployment name — your fullname helper decides it; don't
# guess it. The `app.kubernetes.io/name=lab10-app` label matches both web and echo
# (they share the chart), so filter on the `-web` suffix in the name to get only web:
WEB=$(kubectl get deploy -n lab11 -l app.kubernetes.io/name=lab10-app \
       -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep -- '-web$' | head -1)
kubectl exec -n lab11 deploy/"$WEB" -- env | grep '^DB_'
# DB_USERNAME=app
# DB_PASSWORD=YOUR-PASSWORD                       # ← present in pod env
kubectl describe pod -n lab11 -l app.kubernetes.io/name=lab10-app | grep -A1 -i secret
# (the Secret name appears; the value does NOT — describe redacts envFrom values)
```

Пояснение: команда `helm upgrade --install` передаёт настоящий пароль только из окружения CLI; далее скрипт находит именно Deployment сервиса `web` (по суффиксу `-web`, потому что чарт общий для `web` и `echo`) и проверяет, что переменные `DB_` оказались в поде, а `kubectl describe` не раскрывает значения.

### 2.4 — Proof of work

2.4 — Доказательство работы

Paste into `docs/LAB11.md`:

Вставь в `docs/LAB11.md`:

- Your `templates/secret.yaml` and the relevant `values.yaml` snippet (placeholder values only — *no real password*)
  * Твой `templates/secret.yaml` и релевантный фрагмент `values.yaml` (только значения-заглушки — *никакого реального пароля*)
- The `helm upgrade ... --set` command you actually ran (you can redact the password to `***`)
  * Команда `helm upgrade ... --set`, которую ты реально запускал (пароль можно заменить на `***`)
- The `kubectl exec ... env | grep ^DB_` output proving the values landed in the container (you can redact the password to `***` here too)
  * Вывод `kubectl exec ... env | grep ^DB_`, доказывающий, что значения попали в контейнер (пароль здесь тоже можно заменить на `***`)
- The 2–3 sentence "how I would do this in CI" answer
  * Ответ из 2–3 предложений на тему «как я сделал бы это в CI»

---

## Task 3 — OpenBao Integration (3 pts)

Задача 3 — Интеграция с OpenBao (3 балла)

This is the headline task. You will bring up OpenBao, learn the **env-var contract** that every later production setup reuses, write a **read-only HCL policy**, and prove the policy by trying to write — and failing.

Это главная задача лабы. Ты поднимешь OpenBao, освоишь **контракт переменных окружения**, который переиспользует любая продакшен-конфигурация, напишешь **HCL-политику только на чтение** и докажешь её работу попыткой записи — которая провалится.

### 3.1 — Bring up the dev server

3.1 — Подними dev-сервер

`YOUR TASK`: start the OpenBao dev server with a **memorable** root token ID (you'll use it to log in). Then export the two environment variables every `bao` command depends on.

ТВОЯ ЗАДАЧА: запусти OpenBao dev-сервер с **запоминающимся** root-токеном (он понадобится для входа). Затем экспортируй две переменные окружения, от которых зависит каждая команда `bao`.

Skeleton (fill the YOUR-TASK markers — and yes, the **point** of the blanks is to make you internalize the contract):

Скелет (заполни маркеры YOUR-TASK — да, **смысл** пропусков именно в том, чтобы ты выучил контракт):

```bash
# Start the dev server in the background. -dev = unsealed + in-memory.
# Redirect output to a log file so server lines don't tangle with your prompt.
bao server -dev -dev-root-token-id=___ >/tmp/bao.log 2>&1 &   # YOUR-TASK: pick a token id (e.g. 'devroot')

# Two env vars EVERY bao command needs:
export BAO_ADDR=___                              # YOUR-TASK: the dev server URL (default port is 8200)
export BAO_TOKEN=___                             # YOUR-TASK: must match the -dev-root-token-id above

# Give the listener ~1s to come up before the first call, or poll bao status.
sleep 1
bao status                                       # Sealed false, Storage Type inmem, Version 2.5.0
```

Пояснение: `-dev` означает незапечатанный сервер с хранилищем в памяти; `BAO_ADDR` указывает адрес API (по умолчанию порт 8200), а `BAO_TOKEN` должен совпадать с `-dev-root-token-id`, иначе команды `bao` получат отказ.

> 🧹 **Stopping the dev server.** It's the most recent background job in your shell — `kill %1` (or `kill $(pgrep -f 'bao server -dev')`) when you're done. Restarting wipes every secret (INMEM); that's the whole point.
>
> 🧹 **Остановка dev-сервера.** Это последняя фоновая задача в твоём шелле — когда закончишь, выполни `kill %1` (или `kill $(pgrep -f 'bao server -dev')`). Перезапуск стирает все секреты (INMEM); в этом и смысл.

> 💡 **The contract:** every `bao` (or `vault`) client — your CLI, your apps, the ESO provider, the Agent injector — authenticates by `BAO_ADDR` + a token. In dev mode the token is the root token. In production the token comes from an **auth method** (Kubernetes ServiceAccount JWT, AppRole, OIDC), is short-lived, and is scoped by a policy. The env-var names are the same.
>
> 💡 **Контракт:** любой клиент `bao` (или `vault`) — твой CLI, приложения, провайдер ESO, инжектор Agent — аутентифицируется парой `BAO_ADDR` + токен. В dev-режиме токен — это root-токен. В продакшене токен выдаёт **метод аутентификации** (JWT ServiceAccount Kubernetes, AppRole, OIDC), он короткоживущий и ограничен политикой. Имена переменных окружения при этом те же.

> ⚠️ **Dev server is INMEM.** Every secret you put in goes away when the process restarts. That's deliberate — it makes the lab fast and the security model unmistakable.
>
> ⚠️ **Dev-сервер работает в INMEM.** Каждый положенный туда секрет исчезает при перезапуске процесса. Это сделано намеренно — так лаба проходит быстро, а модель безопасности становится очевидной.

### 3.2 — Put and get a secret on the KV-v2 path

3.2 — Запиши и прочитай секрет по пути KV-v2

`YOUR TASK`: enable the KV-v2 secrets engine at a path you choose, then write a secret containing **at least two** key-value pairs at a path you choose, then read one of the fields back.

ТВОЯ ЗАДАЧА: включи движок секретов KV-v2 по выбранному тобой пути, затем запиши секрет **как минимум с двумя** парами ключ-значение по выбранному пути и прочитай одно из полей обратно.

Skeleton:

Скелет:

```bash
# YOUR TASK lines below:
#   -path=:  pick the mount path (convention: 'secret')
#   put arg: the mount path you chose + a logical name (e.g. secret/lab11/db)
#   first  field=value: a username (use a literal — never a real password)
#   second field=value: a password (literal, not from env)
#   get:     read back one field by name

bao secrets enable -path=___ kv-v2

bao kv put ___/lab11/db \
  ___=app \
  ___=hunter2

bao kv get -field=___ ___/lab11/db
```

Пояснение: `bao secrets enable -path=... kv-v2` включает движок по указанному mount-пути, `bao kv put` записывает пару логин/пароль, а `bao kv get -field=...` возвращает одно поле.

> ⚠️ **KV-v1 vs KV-v2 path gotcha.** `bao kv put secret/foo k=v` writes the *logical* path `secret/foo`, but the **API** path under KV-v2 is `secret/data/foo` (with a `data/` segment inserted). Policies and the Agent injector reference the API path — so an HCL rule must say `path "secret/data/lab11/db"`, not `path "secret/lab11/db"`. Forgetting this is the #1 way a "correct" policy denies a read.
>
> ⚠️ **Ловушка пути KV-v1 против KV-v2.** `bao kv put secret/foo k=v` пишет *логический* путь `secret/foo`, но **API-путь** в KV-v2 — `secret/data/foo` (добавляется сегмент `data/`). Политики и инжектор Agent ссылаются на API-путь — поэтому правило HCL должно выглядеть как `path "secret/data/lab11/db"`, а не `path "secret/lab11/db"`. Забыть об этом — способ номер один, которым «правильная» политика отклоняет чтение.

> ⚠️ **`bao kv put` vs `bao write` on the same data path.** KV-v2 versions data under `data/`; `bao write secret/data/foo data='{"k":"v"}'` works but uses a different JSON shape than `bao kv put`. Stick with `bao kv put` / `bao kv get` for KV-v2; `bao write` for everything else (auth backends, policies, roles).
>
> ⚠️ **`bao kv put` против `bao write` на одном и том же data-пути.** KV-v2 версионирует данные под `data/`; `bao write secret/data/foo data='{"k":"v"}'` работает, но использует другую JSON-форму, чем `bao kv put`. Для KV-v2 держись `bao kv put` / `bao kv get`; `bao write` — для всего остального (бэкенды аутентификации, политики, роли).

### 3.3 — Write the read-only policy

3.3 — Напиши политику только на чтение

`YOUR TASK`: create `k8s/secrets/lab11-read.hcl` with one stanza that grants **read** on the exact data path for your secret, and **nothing else**. No `list`, no `update`, no `delete`, no glob — the whole point is least privilege.

ТВОЯ ЗАДАЧА: создай `k8s/secrets/lab11-read.hcl` с одним блоком, который даёт **чтение** ровно по data-пути твоего секрета и **больше ничего**. Ни `list`, ни `update`, ни `delete`, ни глобов — весь смысл в наименьших привилегиях.

Skeleton (fill the YOUR-TASK markers):

Скелет (заполни маркеры YOUR-TASK):

```hcl
# k8s/secrets/lab11-read.hcl
path "___" {                                     # YOUR-TASK: the API path to the secret you wrote in 3.2
  capabilities = [___]                           # YOUR-TASK: a single-element list, the minimum verb
}
```

Пояснение: в `path` указывается API-путь с сегментом `data/`, а `capabilities` — список из одного минимально необходимого действия, `read`.

Apply it and mint a token bound to it:

Примени её и выпусти привязанный к ней токен:

```bash
bao policy write lab11-read k8s/secrets/lab11-read.hcl
APP_TOKEN=$(bao token create -policy=lab11-read -ttl=1h -field=token)
echo "$APP_TOKEN"      # save for the next step
```

Пояснение: `bao policy write` загружает HCL-политику на сервер, а `bao token create` выпускает токен, ограниченный этой политикой и живущий один час.

### 3.4 — Prove the policy by trying to break it

3.4 — Докажи политику, пытаясь её нарушить

`YOUR TASK`: log in as the new token (set `BAO_TOKEN=$APP_TOKEN`), then run **two** commands and capture both outputs:

ТВОЯ ЗАДАЧА: войди под новым токеном (установи `BAO_TOKEN=$APP_TOKEN`), затем выполни **две** команды и зафиксируй оба вывода:

1. A `bao kv get -field=...` that **succeeds** (the policy allows read).
   * `bao kv get -field=...`, который **успешно** выполняется (политика разрешает чтение).
2. A `bao kv put ...` to the same path that **fails** with a 403 (the policy does not allow write).
   * `bao kv put ...` по тому же пути, который **падает** с 403 (политика не разрешает запись).

The second command failing is the evidence. If it succeeds, your policy granted too much — go back to 3.3.

Именно падение второй команды и есть доказательство. Если она проходит — твоя политика даёт слишком много прав, возвращайся к 3.3.

Skeleton:

Скелет:

```bash
BAO_TOKEN=$APP_TOKEN bao kv get -field=___ ___/lab11/db    # YOUR-TASK: should print the value
BAO_TOKEN=$APP_TOKEN bao kv put ___/lab11/db ___=evil      # YOUR-TASK: should fail with 403
# Error writing data to secret/data/lab11/db: ...permission denied
```

Пояснение: первая команда должна вернуть значение поля, а вторая — завершиться ошибкой `permission denied`, потому что политика разрешает только чтение.

Restore your root token for any cleanup: `export BAO_TOKEN=<your-dev-root-token-id>`.

Верни root-токен для любой последующей очистки: `export BAO_TOKEN=<your-dev-root-token-id>`.

### 3.5 — Proof of work

3.5 — Доказательство работы

Paste into `docs/LAB11.md`:

Вставь в `docs/LAB11.md`:

- `bao version` (must show 2.5.0)
  * `bao version` (должен показать 2.5.0)
- The exact `bao server -dev -dev-root-token-id=...` line and the `export BAO_ADDR`/`BAO_TOKEN` lines (the env-var contract — redact the token if you want)
  * Точная строка `bao server -dev -dev-root-token-id=...` и строки `export BAO_ADDR`/`BAO_TOKEN` (контракт переменных окружения — токен при желании можно скрыть)
- `bao status` showing `Sealed false`, `Storage Type inmem`, `Version 2.5.0`
  * `bao status` с `Sealed false`, `Storage Type inmem`, `Version 2.5.0`
- `bao kv put ...` create confirmation + `bao kv get -field=...` returning your value
  * Подтверждение создания `bao kv put ...` + `bao kv get -field=...`, вернувший твоё значение
- Contents of `k8s/secrets/lab11-read.hcl` (your real policy)
  * Содержимое `k8s/secrets/lab11-read.hcl` (твоя настоящая политика)
- The **two** outputs from 3.4 side by side — the read succeeding and the write failing with `permission denied`. **This is the headline evidence.**
  * **Два** вывода из 3.4 рядом — успешное чтение и неудачная запись с `permission denied`. **Это главное доказательство.**

---

## Task 4 — Documentation (2 pts)

Задача 4 — Документация (2 балла)

`YOUR TASK`: finalize `docs/LAB11.md` with these sections, in this order:

ТВОЯ ЗАДАЧА: заверши `docs/LAB11.md` со следующими разделами в таком порядке:

1. **The base64 'aha' moment** — your decoded plaintext (1.2) + the "why base64 then?" paragraph (1.3)
   * **Момент озарения с base64** — твой декодированный открытый текст (1.2) + абзац «зачем тогда base64?» (1.3)
2. **etcd encryption-at-rest** — the paragraph from 1.3 covering KMS vs `aescbc` and what `EncryptionConfiguration` does not protect
   * **Шифрование etcd при хранении** — абзац из 1.3 про KMS против `aescbc` и про то, от чего `EncryptionConfiguration` не защищает
3. **Helm-managed secret** — chart snippet, install command (redacted), pod env proof (2.4)
   * **Секрет под управлением Helm** — фрагмент чарта, команда установки (со скрытым паролем), доказательство переменных окружения в поде (2.4)
4. **OpenBao workflow** — env-var contract, KV-v2 put/get, the read-only policy + the denied-write proof (3.5)
   * **Работа с OpenBao** — контракт переменных окружения, put/get в KV-v2, политика только на чтение + доказательство отказа в записи (3.5)
5. **Vault → OpenBao history** — one paragraph: BSL Aug 2023, LF fork, why OpenBao for this course
   * **История Vault → OpenBao** — один абзац: BSL в августе 2023, форк от Linux Foundation, почему в этом курсе выбран OpenBao
6. **Security analysis** — when native K8s Secret is fine vs when you need OpenBao; what dev mode hides from you that production exposes (no Raft, no audit device, no auto-unseal, no TLS); 2–3 sentences on the SOPS / Sealed Secrets alternatives and when they make sense
   * **Анализ безопасности** — когда нативного K8s Secret достаточно, а когда нужен OpenBao; что dev-режим скрывает от тебя, а продакшен обнажает (нет Raft, нет audit device, нет auto-unseal, нет TLS); 2–3 предложения об альтернативах SOPS / Sealed Secrets и о том, когда они уместны
7. **Production checklist** — five bullets you would actually put in a prod hardening doc (least-privilege roles, audit device, etc.)
   * **Продакшен-чек-лист** — пять пунктов, которые ты действительно включил бы в документ по усилению безопасности прода (роли с наименьшими привилегиями, audit device и т. д.)

---

## Bonus Task — Pick ONE (2 pts)

Бонусная задача — выбери ОДИН вариант (2 балла)

Pick **A** or **B**. Both are 2 pts. **Do not do both.** Less hand-holding here on purpose — you've earned it.

Выбери **A** или **B**. Оба стоят 2 балла. **Не делай оба.** Здесь намеренно меньше подсказок — ты их заслужил.

### Option A — OpenBao Agent Injector

Вариант A — OpenBao Agent Injector

You install the Agent Injector via the OpenBao Helm chart, annotate a pod, and prove that the secret lands on disk inside the pod without the app knowing about OpenBao.

Ты устанавливаешь Agent Injector через Helm-чарт OpenBao, добавляешь аннотации к поду и доказываешь, что секрет появляется на диске внутри пода, при этом приложение ничего не знает об OpenBao.

**Required artifacts** (in `k8s/secrets/injector.yaml`):

**Обязательные артефакты** (в `k8s/secrets/injector.yaml`):

1. The **annotations** on your pod template that trigger injection. The names below are the contract — fill in the values; the names that end in `-<file>` are templated by *you* (the filename portion becomes the filename under `/vault/secrets/`):
   * **Аннотации** к шаблону пода, запускающие инъекцию. Имена ниже — это контракт; подставь значения; имена, оканчивающиеся на `-<file>`, шаблонизируешь *ты* (часть с именем файла становится именем файла в `/vault/secrets/`):

   ```yaml
   # k8s/secrets/injector.yaml — annotations: section of your pod template
   vault.hashicorp.com/agent-inject: "true"
   vault.hashicorp.com/role: YOUR-TASK                          # the OpenBao role you create below
   vault.hashicorp.com/agent-inject-secret-YOUR-TASK: YOUR-TASK # filename : the KV API path (mind the data/ segment)
   vault.hashicorp.com/agent-inject-template-YOUR-TASK: |       # render the secret as a .env-style file (>=2 keys)
     YOUR-TASK
   ```

   Пояснение: аннотации задают роль и путь к секрету в API; суффикс после `agent-inject-secret-` становится именем файла в `/vault/secrets/`.

2. `serviceAccountName: lab11-sa` on the pod spec — the ServiceAccount whose JWT OpenBao validates.
   * `serviceAccountName: lab11-sa` в спецификации пода — это ServiceAccount, чей JWT проверяет OpenBao.

**Required OpenBao config** (run inside the openbao server pod):

**Требуемая конфигурация OpenBao** (выполняется внутри пода сервера openbao):

- `bao auth enable kubernetes` and `bao write auth/kubernetes/config kubernetes_host=...`
  * `bao auth enable kubernetes` и `bao write auth/kubernetes/config kubernetes_host=...`
- A `bao write auth/kubernetes/role/lab11 ...` that binds `lab11-sa`, namespace `lab11`, the `lab11-read` policy, and a sensible `ttl`.
  * `bao write auth/kubernetes/role/lab11 ...`, привязывающая `lab11-sa`, namespace `lab11`, политику `lab11-read` и разумный `ttl`.

**Proof of work:**

**Доказательство работы:**

- `kubectl get pods -n openbao` showing the injector pod Running
  * `kubectl get pods -n openbao`, показывающий под инжектора в состоянии Running
- `kubectl describe pod -n lab11 <your-pod>` showing the **injected init container + sidecar** (`vault-agent-init` and `vault-agent`) that you did NOT put there
  * `kubectl describe pod -n lab11 <your-pod>`, показывающий **внедрённые init-контейнер и sidecar** (`vault-agent-init` и `vault-agent`), которых ты НЕ добавлял
- `kubectl exec -n lab11 <your-pod> -c <app-container> -- cat /vault/secrets/db` showing the rendered file (redact the value if you like — the *existence* of the file is what matters)
  * `kubectl exec -n lab11 <your-pod> -c <app-container> -- cat /vault/secrets/db`, показывающий отрендеренный файл (значение при желании скрой — важно *существование* файла)

> The injector is a **MutatingAdmissionWebhook**. It edits your podspec at admission time. That's why your YAML has *one* container and the running pod has *three* — the webhook added two. This is the same mechanism Istio uses for its envoy sidecar.
>
> Инжектор — это **MutatingAdmissionWebhook**. Он редактирует podspec в момент admission. Поэтому в твоём YAML *один* контейнер, а в запущенном поде *три* — вебхук добавил два. Тот же механизм Istio использует для своего sidecar envoy.

### Option B — External Secrets Operator (ESO)

Вариант B — External Secrets Operator (ESO)

You install ESO via Helm, point a `SecretStore` at your OpenBao server, and an `ExternalSecret` produces a **native K8s Secret** that your app consumes with `envFrom: secretRef`. The app never knows ESO exists.

Ты устанавливаешь ESO через Helm, направляешь `SecretStore` на свой сервер OpenBao, и `ExternalSecret` создаёт **нативный K8s Secret**, который потребляет твоё приложение через `envFrom: secretRef`. Приложение вообще не знает о существовании ESO.

**Required artifacts** (in `k8s/secrets/eso.yaml`):

**Обязательные артефакты** (в `k8s/secrets/eso.yaml`):

1. A `SecretStore` CRD (`apiVersion: external-secrets.io/v1`) named `openbao` in namespace `lab11`, with a `provider.vault` block (ESO's `vault` provider is compatible with OpenBao). It uses `kubernetes` auth, the role you create in OpenBao, and `serviceAccountRef: {name: lab11-sa}`.
   * CRD `SecretStore` (`apiVersion: external-secrets.io/v1`) с именем `openbao` в namespace `lab11` и блоком `provider.vault` (провайдер `vault` в ESO совместим с OpenBao). Он использует аутентификацию `kubernetes`, роль, созданную тобой в OpenBao, и `serviceAccountRef: {name: lab11-sa}`.
2. An `ExternalSecret` CRD producing a native `db-creds` Secret.
   * CRD `ExternalSecret`, создающий нативный Secret `db-creds`.

Skeleton (fill the YOUR-TASK markers — the CRD field names are the contract):

Скелет (заполни маркеры YOUR-TASK — имена полей CRD здесь и есть контракт):

```yaml
# k8s/secrets/eso.yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata: {name: openbao, namespace: lab11}
spec:
  provider:
    vault:
      server: YOUR-TASK                 # the in-cluster URL of the OpenBao server (Service DNS + port)
      path: YOUR-TASK                   # the KV mount path you chose in Task 3.2
      version: YOUR-TASK                # KV engine version
      auth:
        kubernetes:
          mountPath: kubernetes
          role: YOUR-TASK               # the OpenBao role bound to lab11-sa
          serviceAccountRef: {name: lab11-sa}
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: db, namespace: lab11}
spec:
  refreshInterval: YOUR-TASK            # production default vs demo: pick deliberately
  secretStoreRef: {name: openbao, kind: SecretStore}
  target: {name: YOUR-TASK}             # the native K8s Secret ESO will produce (consumed by your app)
  data:
    - secretKey: YOUR-TASK              # the key as it appears IN the produced K8s Secret
      remoteRef: {key: YOUR-TASK, property: YOUR-TASK}  # the OpenBao logical path + the field name in it
    # YOUR-TASK: add a second key — username — with the same shape
```

Пояснение: `SecretStore` описывает, где и как ESO аутентифицируется в OpenBao, а `ExternalSecret` — какие поля оттуда взять и в какой нативный Secret их положить; `refreshInterval` задаёт период опроса.

**Required OpenBao config:** same Kubernetes auth + role binding as Option A (it's the same auth method).

**Требуемая конфигурация OpenBao:** та же аутентификация Kubernetes и привязка роли, что в варианте A (метод аутентификации один и тот же).

**Required app wiring:** edit your Lab 10 chart to consume `db-creds` via `envFrom: secretRef`. Note that the app is now reading a *native K8s Secret* — no sidecar, no annotations, no OpenBao client in the pod. ESO does the syncing out-of-band.

**Требуемая интеграция приложения:** отредактируй чарт из лабы 10, чтобы он потреблял `db-creds` через `envFrom: secretRef`. Обрати внимание: приложение теперь читает *нативный K8s Secret* — ни sidecar, ни аннотаций, ни клиента OpenBao в поде. Синхронизацию выполняет ESO вне репозитория.

**Proof of work:**

**Доказательство работы:**

- `kubectl get pods -n external-secrets` showing the ESO controller Running
  * `kubectl get pods -n external-secrets`, показывающий контроллер ESO в состоянии Running
- `kubectl get externalsecret db -n lab11` showing `STATUS: SecretSynced`
  * `kubectl get externalsecret db -n lab11` с `STATUS: SecretSynced`
- `kubectl get secret db-creds -n lab11` showing the ESO-produced Secret exists
  * `kubectl get secret db-creds -n lab11`, показывающий, что созданный ESO Secret существует
- `kubectl exec -n lab11 deploy/lab10-app-web -- env | grep ^DB_` showing the values landed in the app
  * `kubectl exec -n lab11 deploy/lab10-app-web -- env | grep ^DB_`, показывающий, что значения попали в приложение

> ESO **polls** OpenBao every `refreshInterval`. Lower it for the demo (e.g. `30s`), but `1h` is the production default — rotating in OpenBao does not propagate instantly. That trade-off is the headline difference vs the Agent Injector (which renews via the token lease).
>
> ESO **опрашивает** OpenBao каждые `refreshInterval`. Для демо уменьши его (например, `30s`), но `1h` — продакшен-значение по умолчанию: ротация в OpenBao не распространяется мгновенно. Этот компромисс и есть главное отличие от Agent Injector (который обновляет секрет через аренду токена).

**Bonus documentation:** append a section to `docs/LAB11.md` with the manifest you used, the proof captures above, and **one paragraph** on why you picked A or B for the kind of app you'd want to run with it (think: "dynamic DB creds with 15-min TTL" → Agent Injector; "every microservice in a 50-pod monorepo reads the same handful of secrets" → ESO).

**Документация по бонусу:** добавь в `docs/LAB11.md` раздел с использованным манифестом, снятыми доказательствами выше и **одним абзацем** о том, почему ты выбрал A или B для приложения, с которым хотел бы это запускать (например: «динамические учётные данные БД с TTL 15 минут» → Agent Injector; «каждый микросервис в монорепозитории из 50 подов читает один и тот же набор секретов» → ESO).

---

## How to Submit

Как сдать

```bash
git switch -c lab11
git add k8s/secrets/ k8s/lab10-app/templates/secret.yaml \
        k8s/lab10-app/values.yaml docs/LAB11.md
# also stage k8s/secrets/injector.yaml (Option A) OR k8s/secrets/eso.yaml (Option B) if you did the bonus
git commit -m "feat(lab11): secrets + OpenBao (read-only policy, base64 demo)"
git push -u origin lab11
```

Пояснение: создаётся ветка `lab11`, в коммит попадают только манифесты, чарт и отчёт — файлы с реальными секретами не добавляются.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab11` → `course-repo:master` *(reviewed)*
  * `your-fork:lab11` → `course-repo:master` *(проверяется ревьюером)*
- `your-fork:lab11` → `your-fork:master`
  * `your-fork:lab11` → `your-fork:master`

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 — app-credentials.yaml + base64 decode proof in docs/LAB11.md
- [ ] Task 2 — templates/secret.yaml in chart, placeholders only in values.yaml, env proof
- [ ] Task 3 — bao 2.5.0 dev server, BAO_ADDR/BAO_TOKEN exported, KV put/get, read-only policy, denied-write proof
- [ ] Task 4 — docs/LAB11.md has all 7 sections
- [ ] Bonus (optional) — Agent Injector OR ESO with proof captures
- [ ] No real secret values in any committed file (grep your diff before pushing)
```

Пояснение: это чек-лист самопроверки перед отправкой PR — сверь его перед пушем.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (2 pts)

Задача 1 (2 балла)

- ✅ `k8s/secrets/app-credentials.yaml` is your own hand-written `kind: Secret` with a deliberate `type:` choice and two base64-encoded `data:` keys
  * ✅ `k8s/secrets/app-credentials.yaml` — твой собственный написанный вручную `kind: Secret` с осознанным выбором `type:` и двумя ключами `data:` в base64
- ✅ `docs/LAB11.md` contains the base64-decode one-liner **with your own plaintext output**
  * ✅ В `docs/LAB11.md` есть однострочник декодирования base64 **с твоим собственным открытым выводом**
- ✅ "Why base64?" answer and etcd-at-rest paragraph are present and correct
  * ✅ Ответ «Зачем base64?» и абзац про etcd-at-rest присутствуют и корректны

### Task 2 (3 pts)

Задача 2 (3 балла)

- ✅ `k8s/lab10-app/templates/secret.yaml` exists, uses the chart's name + label helpers, and reads from `.Values.secret.*`
  * ✅ `k8s/lab10-app/templates/secret.yaml` существует, использует хелперы имени и лейблов чарта и читает из `.Values.secret.*`
- ✅ `values.yaml` ships **placeholder** strings only — no real password
  * ✅ В `values.yaml` только **строки-заглушки** — никакого реального пароля
- ✅ Deployment consumes the Secret via `envFrom: secretRef`
  * ✅ Deployment потребляет Secret через `envFrom: secretRef`
- ✅ Pod env confirmed via `kubectl exec ... env | grep ^DB_`; `kubectl describe` does NOT show the values
  * ✅ Переменные окружения пода подтверждены через `kubectl exec ... env | grep ^DB_`; `kubectl describe` НЕ показывает значения
- ✅ `docs/LAB11.md` documents the CI pattern you'd use to avoid `--set` leaking the password
  * ✅ `docs/LAB11.md` описывает CI-паттерн, который не даст `--set` утечь пароль

### Task 3 (3 pts)

Задача 3 (3 балла)

- ✅ `bao version` shows 2.5.0
  * ✅ `bao version` показывает 2.5.0
- ✅ `bao server -dev -dev-root-token-id=...` was run; `BAO_ADDR` and `BAO_TOKEN` exported
  * ✅ Выполнена команда `bao server -dev -dev-root-token-id=...`; `BAO_ADDR` и `BAO_TOKEN` экспортированы
- ✅ KV-v2 enabled, secret written with ≥ 2 keys, read back with `bao kv get -field=...`
  * ✅ KV-v2 включён, секрет записан с ≥ 2 ключами и прочитан через `bao kv get -field=...`
- ✅ `k8s/secrets/lab11-read.hcl` grants **read only** on the exact data path (no glob, no extra capabilities)
  * ✅ `k8s/secrets/lab11-read.hcl` даёт **только чтение** ровно по data-пути (без глобов и лишних возможностей)
- ✅ The denied-write evidence (`bao kv put ...` → `permission denied`) is captured in `docs/LAB11.md`
  * ✅ Доказательство отказа в записи (`bao kv put ...` → `permission denied`) сохранено в `docs/LAB11.md`

### Task 4 (2 pts)

Задача 4 (2 балла)

- ✅ `docs/LAB11.md` has all seven sections (1.2 + 1.3, etcd-at-rest, Helm, OpenBao, Vault→OpenBao history, security analysis, production checklist)
  * ✅ В `docs/LAB11.md` есть все семь разделов (1.2 + 1.3, etcd-at-rest, Helm, OpenBao, история Vault→OpenBao, анализ безопасности, продакшен-чек-лист)

### Bonus (2 pts) — one option only

Бонус (2 балла) — только один вариант

- ✅ **A:** Agent Injector deployed; injected init+sidecar visible in `kubectl describe pod`; `/vault/secrets/<file>` exists in the app container; custom template renders multi-key `.env`
  * ✅ **A:** Agent Injector развёрнут; внедрённые init- и sidecar-контейнеры видны в `kubectl describe pod`; файл `/vault/secrets/<file>` существует в контейнере приложения; кастомный шаблон рендерит `.env` с несколькими ключами
- ✅ **B:** ESO installed; `ExternalSecret` shows `SecretSynced`; `db-creds` native Secret exists; app pod env contains the values
  * ✅ **B:** ESO установлен; `ExternalSecret` показывает `SecretSynced`; нативный Secret `db-creds` существует; переменные окружения пода приложения содержат значения

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — K8s Secrets & base64 trap | **2** | Hand-written Secret, decoded plaintext, base64 vs encryption explained, etcd-at-rest understood |
| **Task 2** — Helm-managed Secret | **3** | Templated Secret, placeholder values, envFrom wiring, real value injected at install, CI pattern documented |
| **Task 3** — OpenBao integration | **3** | Dev server up, env-var contract internalized, KV-v2 put/get, read-only policy enforces (denied-write proof) |
| **Task 4** — Documentation | **2** | All seven sections present, security analysis genuine, BSL/OpenBao history correct |
| **Bonus** — Injector OR ESO | **2** | Working injection (A) or sync (B) with manifests + proof |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — K8s Secrets и ловушка base64 | **2** | Secret написан вручную, открытый текст декодирован, объяснена разница base64 и шифрования, понято шифрование etcd-at-rest |
| **Задача 2** — Secret под управлением Helm | **3** | Шаблонизированный Secret, значения-заглушки, интеграция через envFrom, реальное значение передано при установке, описан CI-паттерн |
| **Задача 3** — интеграция с OpenBao | **3** | Dev-сервер поднят, контракт переменных окружения усвоен, put/get в KV-v2, политика только на чтение работает (доказан отказ в записи) |
| **Задача 4** — документация | **2** | Все семь разделов на месте, анализ безопасности содержательный, история BSL/OpenBao изложена верно |
| **Бонус** — Injector ИЛИ ESO | **2** | Работает инъекция (A) или синхронизация (B), есть манифесты и доказательства |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading bands:**

**Диапазоны оценок:**

- **10/10:** All four main tasks done, the denied-write evidence is real, no real secrets committed, security analysis shows genuine understanding
  * **10/10:** все четыре основные задачи выполнены, доказательство отказа в записи реальное, настоящие секреты не закоммичены, анализ безопасности показывает искреннее понимание
- **8–9/10:** All tasks done with minor gaps in docs or one missing proof capture
  * **8–9/10:** все задачи выполнены с мелкими пробелами в документации или без одного доказательства
- **6–7/10:** Tasks 1+2 solid, OpenBao up but policy is too permissive (no denied-write evidence) or KV path wrong
  * **6–7/10:** задачи 1 и 2 сделаны уверенно, OpenBao поднят, но политика слишком разрешающая (нет доказательства отказа в записи) или путь KV неверный
- **<6/10:** OpenBao not running, base64 demo missing, or real secrets committed
  * **<6/10:** OpenBao не запущен, демонстрация base64 отсутствует или закоммичены настоящие секреты

---

## Resources

Ресурсы

<details>
<summary>📚 Documentation</summary>

📚 Документация

- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — concepts + the `type:` table
  * [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — концепции + таблица `type:`
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) — EncryptionConfiguration, KMS provider, key rotation
  * [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) — EncryptionConfiguration, провайдер KMS, ротация ключей
- [OpenBao docs](https://openbao.org/docs/) — `bao` CLI, KV-v2, auth methods, policies
  * [Документация OpenBao](https://openbao.org/docs/) — CLI `bao`, KV-v2, методы аутентификации, политики
- [OpenBao 2.5.0 release notes](https://openbao.org/community/release-notes/2-5-0/) — Feb 4 2026
  * [Release notes OpenBao 2.5.0](https://openbao.org/community/release-notes/2-5-0/) — 4 февраля 2026
- [OpenBao Kubernetes auth](https://openbao.org/docs/auth/kubernetes/) — TokenReview API, role binding
  * [Kubernetes auth в OpenBao](https://openbao.org/docs/auth/kubernetes/) — TokenReview API, привязка роли
- [Vault Agent annotations (OpenBao-compatible)](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations) — annotation reference
  * [Аннотации Vault Agent (совместимы с OpenBao)](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations) — справочник по аннотациям
- [External Secrets Operator](https://external-secrets.io/) — CRDs + provider list
  * [External Secrets Operator](https://external-secrets.io/) — CRD + список провайдеров
- [ESO Vault/OpenBao provider](https://external-secrets.io/latest/provider/hashicorp-vault/) — config shape
  * [Провайдер Vault/OpenBao в ESO](https://external-secrets.io/latest/provider/hashicorp-vault/) — форма конфигурации

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **Base64 is encoding, not encryption.** `kubectl get secret -o yaml | grep -A1 data:` followed by `| base64 -d` recovers plaintext with **no key** — the #1 fact every junior gets wrong. If you walk away with one thing from this lab, walk away with this.
  * **Base64 — это кодирование, а не шифрование.** `kubectl get secret -o yaml | grep -A1 data:` с последующим `| base64 -d` возвращает открытый текст **без всякого ключа** — факт номер один, который путают все новички. Если из этой лабы ты вынесешь одну вещь, пусть это будет она.
- **`echo` adds a trailing newline.** `echo "foo" | base64` ≠ `echo -n "foo" | base64`. The newline silently invalidates passwords inside the app. Use `-n`.
  * **`echo` добавляет перевод строки в конце.** `echo "foo" | base64` ≠ `echo -n "foo" | base64`. Перевод строки молча ломает пароли внутри приложения. Используй `-n`.
- **etcd encryption-at-rest needs KMS, not just `aescbc`.** Local-key `aescbc` providers store the key on the same disk as the data they "protect" — the key is sitting next to the lock. KMS providers (AWS KMS, GCP KMS, OpenBao Transit) keep the key out of the cluster.
  * **Шифрование etcd при хранении требует KMS, а не только `aescbc`.** Локальные ключи в провайдере `aescbc` лежат на том же диске, что и данные, которые они «защищают» — ключ рядом с замком. KMS-провайдеры (AWS KMS, GCP KMS, OpenBao Transit) держат ключ вне кластера.
- **OpenBao dev server is INMEM.** Restart the process and every secret is gone. That's the point — dev mode is for learning the API, not for storing anything. Read the lecture's anti-pattern slide on running dev mode in prod.
  * **Dev-сервер OpenBao работает в INMEM.** Перезапусти процесс — и все секреты исчезнут. В этом и смысл: dev-режим нужен для изучения API, а не для хранения чего-либо. Перечитай слайд лекции об антипаттерне запуска dev-режима в проде.
- **KV-v1 vs KV-v2 path difference.** `bao kv put secret/foo k=v` writes the *logical* path; policies and the Agent injector must reference the **API** path (`secret/data/foo`, with `data/` inserted) or the read silently denies. Same gotcha lives in the ESO `remoteRef.key` field.
  * **Разница путей KV-v1 и KV-v2.** `bao kv put secret/foo k=v` пишет *логический* путь; политики и инжектор Agent должны ссылаться на **API**-путь (`secret/data/foo`, с вставленным `data/`), иначе чтение молча отклоняется. Та же ловушка живёт в поле `remoteRef.key` у ESO.
- **`bao kv put` vs `bao write`.** Both touch the same path on KV-v2 but use different JSON shapes; stick to `kv put`/`kv get` for KV-v2 to avoid the shape mismatch.
  * **`bao kv put` против `bao write`.** Обе команды трогают один и тот же путь в KV-v2, но используют разные JSON-формы; держись `kv put`/`kv get` для KV-v2, чтобы избежать несовпадения форм.
- **Helm `--set` leaks to CI logs.** `helm upgrade ... --set db.password='hunter2'` in a GitHub Action log is a Code Spaces / tj-actions waiting to happen. Wrap as `${{ secrets.* }}` and reference by env var; better, use a real secret manager and let ESO/Agent Injector deliver the value.
  * **Helm `--set` утекает в логи CI.** `helm upgrade ... --set db.password='hunter2'` в логе GitHub Action — это готовый инцидент в духе Code Spaces / tj-actions. Оборачивай в `${{ secrets.* }}` и передавай через переменную окружения; ещё лучше — используй настоящий менеджер секретов и отдай доставку значения ESO или Agent Injector.
- **`automountServiceAccountToken: true` by default.** Every pod gets a SA token whether it needs one or not. For pods that don't talk to the K8s API or OpenBao, set this to `false` to shrink the blast radius.
  * **`automountServiceAccountToken: true` по умолчанию.** Каждый под получает токен ServiceAccount, нужен он ему или нет. Для подов, которые не обращаются к API K8s или OpenBao, выставь `false`, чтобы уменьшить радиус поражения.
- **`vault.hashicorp.com/role: admin` is the new `--privileged`.** Bind one role per app per environment. Least privilege is the entire point of the policy you wrote in 3.3.
  * **`vault.hashicorp.com/role: admin` — это новый `--privileged`.** Привязывай одну роль на приложение на окружение. Наименьшие привилегии — весь смысл политики, которую ты написал в 3.3.
- **Dev-mode root token in your shell history.** `export BAO_TOKEN=root` survives in `~/.bash_history`. Use a unique value per dev session and clear it (`unset BAO_TOKEN`) when you're done.
  * **Root-токен dev-режима в истории шелла.** `export BAO_TOKEN=root` останется в `~/.bash_history`. Используй уникальное значение для каждой dev-сессии и очищай его (`unset BAO_TOKEN`), когда закончишь.

</details>

<details>
<summary>🛠️ Tools worth knowing</summary>

🛠️ Инструменты, которые стоит знать

- [gitleaks](https://github.com/gitleaks/gitleaks) — pre-commit + CI scanner; catches the human moments
  * [gitleaks](https://github.com/gitleaks/gitleaks) — сканер для pre-commit и CI; ловит человеческий фактор
- [trufflehog](https://github.com/trufflesecurity/trufflehog) — deeper detector, finds high-entropy strings in git history
  * [trufflehog](https://github.com/trufflesecurity/trufflehog) — более глубокий детектор, находит высокоэнтропийные строки в истории git
- [SOPS](https://getsops.io/) — file-level encryption with KMS/age/PGP; GitOps-friendly
  * [SOPS](https://getsops.io/) — шифрование на уровне файлов с KMS/age/PGP; дружит с GitOps
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) — per-cluster public-key encryption; CRD + controller
  * [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) — шифрование публичным ключом на кластер; CRD + контроллер
- [k9s](https://k9scli.io/) — terminal UI; `:secret` shows them all at once
  * [k9s](https://k9scli.io/) — терминальный UI; `:secret` показывает все секреты сразу

</details>

---

## Looking Ahead

Что дальше

| Lab | What it adds |
|---:|---|
| 12 | ConfigMaps + PVC — non-sensitive config + persistent state survives pod deletion |
| 13 | ArgoCD GitOps — your secured chart deploys via Application/ApplicationSet |
| 14 | Argo Rollouts canary; secrets rotate underneath progressive delivery |
| 15 | StatefulSets — stable identity + per-pod PVC + secrets per replica |
| 16 | kube-prometheus stack — scrape OpenBao's `/sys/metrics` |

| Лаба | Что добавляет |
|---:|---|
| 12 | ConfigMaps + PVC — несекретная конфигурация + постоянное состояние, переживающее удаление пода |
| 13 | ArgoCD GitOps — твой защищённый чарт разворачивается через Application/ApplicationSet |
| 14 | Argo Rollouts canary; секреты ротируются под канареечным релизом |
| 15 | StatefulSets — стабильная идентичность + PVC на под + секреты на реплику |
| 16 | стек kube-prometheus — сбор `/sys/metrics` из OpenBao |

**Good luck.** 🔐

**Удачи.** 🔐

> **Remember:** the moment you decoded your own Secret with `base64 -d` and got back plaintext is the moment you understood why every breach in the lecture's incident list was preventable. Keep that feeling. Carry it into every PR review where someone "just for now" puts a real value in `values.yaml`.
>
> **Запомни:** момент, когда ты декодировал собственный Secret через `base64 -d` и получил открытый текст, — это момент, когда ты понял, почему каждую утечку из списка инцидентов в лекции можно было предотвратить. Сохрани это ощущение. Проноси его через каждое ревью PR, где кто-то «просто пока» кладёт реальное значение в `values.yaml`.
