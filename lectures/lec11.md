# 📌 Lecture 11 — Secrets & OpenBao: Stop Putting Passwords in Git

Лекция 11 — Секреты и OpenBao: перестань хранить пароли в Git

## 📍 Slide 1 – 🔐 Welcome to Secret Management

Слайд 1 — Добро пожаловать в управление секретами

* 🌍 **Lab 10 left you with a clean Helm chart** — Deployment, Service, Ingress, values per environment. One problem: a real app needs a DB password, an API key, a TLS cert. You can't put those in `values.yaml` and push to GitHub.
  * **Лаба 10 оставила тебе чистый Helm-чарт** — Deployment, Service, Ingress и values для каждого окружения. Одна проблема: реальному приложению нужны пароль к БД, API-ключ и TLS-сертификат. Их нельзя положить в `values.yaml` и запушить в GitHub.
* 😱 **Secrets in code is the #1 cause of cloud breaches** — and the cheapest to prevent.
  * **Секреты в коде — причина облачных утечек номер один**, и предотвратить их дешевле всего.
* 🎯 This lecture: K8s `Secret` (what it actually is), encryption-at-rest, the **OpenBao** project (Vault's open-source successor), and the two patterns to inject secrets into pods — **Vault Agent Injector** and **External Secrets Operator**.
  * Эта лекция: K8s `Secret` (что это на самом деле), шифрование при хранении, проект **OpenBao** (open-source-преемник Vault) и два паттерна внедрения секретов в поды — **Vault Agent Injector** и **External Secrets Operator**.
* 🔗 **Tie-in to Lab 11:** add a `templates/secrets.yaml` to your Lab 10 chart, install **OpenBao** via Helm, configure Kubernetes auth, and inject a secret via Vault Agent annotations.
  * **Связь с лабой 11:** добавь в свой чарт из лабы 10 файл `templates/secrets.yaml`, установи **OpenBao** через Helm, настрой Kubernetes auth и внедри секрет через аннотации Vault Agent.

```mermaid
flowchart LR
  Code[📄 Secrets in code] -->|❌ leak| Breach[💥 Breach]
  Code -->|✅ refactor| K8s[☸️ K8s Secret]
  K8s -->|🔐 encrypt-at-rest| KMS[🔒 KMS / etcd]
  K8s -->|🤖 sync from| Bao[🏰 OpenBao]
  Bao --> Apps[📦 Pods]
```

Схема: секреты в коде ведут к 💥 утечке; после рефакторинга они попадают в ☸️ K8s Secret, шифруются при хранении через 🔒 KMS, синхронизируются из 🏰 OpenBao и доходят до подов.

---

## 📍 Slide 2 – 🎯 Learning Outcomes

Слайд 2 — Результаты обучения

| # | Outcome |
|---|---------|
| 1 | 🧠 Explain why a K8s `Secret` is **not encrypted** by default (it's base64) |
| 2 | 🔐 Enable **encryption-at-rest** in etcd using an `EncryptionConfiguration` + KMS provider |
| 3 | 🏰 Run **OpenBao 2.5.0** in Kubernetes; configure KV-v2 + Kubernetes auth + a read policy |
| 4 | 💉 Use the **Vault Agent Injector** sidecar pattern to render secrets to a file in the pod |
| 5 | 🤖 Use the **External Secrets Operator** (ESO) to sync from OpenBao → native K8s `Secret` |
| 6 | 🗺️ Choose the right pattern (Agent Injector vs ESO vs CSI vs Sealed Secrets vs SOPS) |

| # | Результат |
|---|-----------|
| 1 | 🧠 Объяснить, почему K8s `Secret` **не зашифрован** по умолчанию (это base64) |
| 2 | 🔐 Включить **шифрование при хранении** в etcd с помощью `EncryptionConfiguration` и провайдера KMS |
| 3 | 🏰 Запустить **OpenBao 2.5.0** в Kubernetes; настроить KV-v2, Kubernetes auth и политику на чтение |
| 4 | 💉 Использовать паттерн сайдкара **Vault Agent Injector**, чтобы рендерить секреты в файл внутри пода |
| 5 | 🤖 Использовать **External Secrets Operator** (ESO), чтобы синхронизировать секреты из OpenBao → в нативный K8s `Secret` |
| 6 | 🗺️ Выбрать правильный паттерн (Agent Injector vs ESO vs CSI vs Sealed Secrets vs SOPS) |

**Tech stack pinned for May 2026:** Kubernetes **1.36**, **OpenBao 2.5.0** (Linux Foundation, MPL-2.0), External Secrets Operator **v2.5.0**, `vault` CLI (still works against OpenBao — wire-compatible).

**Технологический стек, зафиксированный на май 2026:** Kubernetes **1.36**, **OpenBao 2.5.0** (Linux Foundation, MPL-2.0), External Secrets Operator **v2.5.0**, CLI `vault` (по-прежнему работает с OpenBao — они совместимы на уровне протокола).

---

## 📍 Slide 3 – 💀 The Cost of Getting It Wrong

Слайд 3 — Цена ошибки

> 💬 *"The only truly secure system is one that is powered off, cast in a block of concrete and sealed in a lead-lined room with armed guards."* — Gene Spafford
>
> 💬 *«По-настоящему безопасна только система, которая выключена, залита бетоном и запечатана в комнате со свинцовыми стенами и вооружённой охраной»*. — Джин Спаффорд

* 💸 **IBM Cost of a Data Breach 2025:** global average **$4.44M** per breach; US record **$10.22M**
  * 💸 **IBM Cost of a Data Breach 2025:** средняя утечка в мире — **$4,44 млн**; рекорд в США — **$10,22 млн**
* 🔍 **Credential abuse** is the #1 initial-access vector in the Verizon DBIR every year since 2019
  * 🔍 **Злоупотребление украденными учётными данными** — вектор первоначального доступа номер один в отчёте Verizon DBIR каждый год с 2019-го
* ⏱️ **Mean time to detect** a credential breach: ~204 days (IBM 2025)
  * ⏱️ **Среднее время обнаружения** утечки учётных данных: около 204 дней (IBM, 2025)
* 🤖 **GitGuardian 2024:** ~12.8M new secrets leaked to public GitHub in a single year — most never rotated
  * 🤖 **GitGuardian 2024:** около 12,8 млн новых секретов утекло в публичный GitHub за один год — большинство так и не заменили

> 🤔 **Think:** how many `.env` files do you have on your laptop right now that contain a real production password?
>
> 🤔 **Подумай:** сколько `.env`-файлов с настоящим продовым паролем лежит прямо сейчас на твоём ноутбуке?

---

## 📍 Slide 4 – 📜 Real Incidents — Learn From Their Pain

Слайд 4 — Реальные инциденты — учись на чужой боли

| Year | Who | What |
|------|-----|------|
| 2014 | **Code Spaces** | AWS root keys in code → attacker wiped all backups + all customer data → **company shut down within 12 hours** |
| 2016 | **Uber** | AWS creds hardcoded in a private GitHub repo (compromised dev account) → S3 dump of **57M users** → **$148M settlement** |
| 2022 | **Dropbox** | 130 private GH repos stolen via phishing — leaked internal API keys + 3rd-party creds |
| 2022 | **Toyota** | `.git` directory served from a public asset → customer-data DB credentials leaked **for 10 years** |
| 2025 | **tj-actions/changed-files** | CVE-2025-30066, compromised popular GitHub Action **dumped every secret** in CI logs of 23,000+ repos |

| Год | Кто | Что |
|------|-----|------|
| 2014 | **Code Spaces** | Ключи AWS root в коде → атакующий стёр все бэкапы и все данные клиентов → **компания закрылась за 12 часов** |
| 2016 | **Uber** | Учётные данные AWS захардкожены в приватном репозитории GitHub (скомпрометирован аккаунт разработчика) → слив из S3 данных **57 млн пользователей** → **урегулирование на $148 млн** |
| 2022 | **Dropbox** | 130 приватных репозиториев GitHub украдены через фишинг — утекли внутренние API-ключи и данные сторонних сервисов |
| 2022 | **Toyota** | Каталог `.git` отдавался как публичный ресурс → учётные данные от БД с данными клиентов утекали **10 лет** |
| 2025 | **tj-actions/changed-files** | CVE-2025-30066, скомпрометированный популярный GitHub Action **слил все секреты** в логи CI более чем 23 000 репозиториев |

> 🔥 **Pattern:** every single one of these was a **secret stored in the wrong place** (code, git, CI logs, public asset). Not zero-days. Not 0-click. Just a credential in the wrong file.
>
> 🔥 **Закономерность:** в каждом без исключения случае **секрет лежал не там, где надо** (код, git, логи CI, публичный ресурс). Это не zero-day и не 0-click. Просто учётные данные не в том файле.

---

## 📍 Slide 5 – ☸️ The Kubernetes `Secret` Object

Слайд 5 — Объект `Secret` в Kubernetes

A `Secret` is a first-class K8s API object, separate from `ConfigMap` because it's *intended* for sensitive data.

`Secret` — полноценный API-объект Kubernetes, отдельный от `ConfigMap`, потому что он *предназначен* для чувствительных данных.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
  namespace: lab11
type: Opaque
data:                                  # ⚠️ values must be base64-encoded
  username: YWRtaW4=
  password: U3VwZXJTZWNyZXQxMjM=
# stringData:                          # ✅ alternative: K8s base64-encodes for you
#   username: admin
#   password: SuperSecret123
```

Пояснение к коду: в `data` значения обязаны быть в base64 (`username: YWRtaW4=` — это `admin`), а в закомментированном `stringData` можно писать открытый текст — Kubernetes закодирует его сам.

* 📦 Three creation patterns: `kubectl create secret` (imperative), YAML `kind: Secret` (declarative), Helm `templates/secrets.yaml`
  * 📦 Три способа создать секрет: `kubectl create secret` (императивно), YAML с `kind: Secret` (декларативно), Helm `templates/secrets.yaml`
* 🔑 **Built-in types:** `Opaque` (generic), `kubernetes.io/dockerconfigjson` (image pulls), `kubernetes.io/tls` (cert + key), `kubernetes.io/service-account-token`
  * 🔑 **Встроенные типы:** `Opaque` (общий), `kubernetes.io/dockerconfigjson` (для получения образов), `kubernetes.io/tls` (сертификат и ключ), `kubernetes.io/service-account-token`
* 🚦 Consumed by pods as **env vars** (`envFrom`/`secretKeyRef`) or as a **file mount** (`projected` / `secret` volume)
  * 🚦 Под получает их как **переменные окружения** (`envFrom`/`secretKeyRef`) или как **смонтированный файл** (том `projected` / `secret`)

---

## 📍 Slide 6 – ⚠️ Base64 Is Not Encryption

Слайд 6 — Base64 — это не шифрование

> ⚠️ **The single most-tested fact in K8s exams** — and the misunderstanding behind every junior breach.
>
> ⚠️ **Самый частый факт на экзаменах по K8s** — и главное недоразумение за каждой утечкой у джунов.

```bash
$ echo "SuperSecret123" | base64
U3VwZXJTZWNyZXQxMjMK
$ echo "U3VwZXJTZWNyZXQxMjMK" | base64 -d
SuperSecret123                         # 🔓 No key. No password. Just decode.
```

Пояснение: `base64 -d` возвращает исходную строку без всякого ключа и пароля — поэтому base64 ничего не защищает.

| 🔄 Encoding (base64) | 🔐 Encryption (AES-GCM, KMS, …) |
|----------------------|----------------------------------|
| ✅ Reversible by anyone with the string | 🔑 Requires a key to decrypt |
| 🎯 Designed for binary-safe text transport | 🛡️ Designed for confidentiality |
| 📦 Same data, different format | 🔒 Different data, mathematically secure |

| 🔄 Кодирование (base64) | 🔐 Шифрование (AES-GCM, KMS, …) |
|----------------------|----------------------------------|
| ✅ Обратимо любым, у кого есть строка | 🔑 Для расшифровки нужен ключ |
| 🎯 Создано для бинарно-безопасной передачи текста | 🛡️ Создано для конфиденциальности |
| 📦 Те же данные, другой формат | 🔒 Другие данные, математически защищённые |

> 🔥 **Why base64 then?** Because `Secret.data` values must be valid JSON/YAML strings — binary like a TLS key wouldn't fit otherwise. Confidentiality is the API server's + etcd's job, not base64's.
>
> 🔥 **Тогда зачем base64?** Потому что значения `Secret.data` должны быть корректными строками JSON/YAML — бинарные данные вроде TLS-ключа иначе туда не поместятся. За конфиденциальность отвечают API-сервер и etcd, а не base64.

---

## 📍 Slide 7 – 🗄️ Where Secrets Actually Live: etcd

Слайд 7 — Где на самом деле хранятся секреты: etcd

```mermaid
flowchart LR
  Pod[📦 Pod] -->|GET secret/db-creds| API[🎯 kube-apiserver]
  API -->|RBAC check| API
  API -->|read| Etcd[(🗄️ etcd<br/>by default: plaintext base64)]
  Etcd -->|return| API
  API -->|inject env/file| Pod
```

Схема: под запрашивает секрет у kube-apiserver, тот проверяет RBAC, читает данные из etcd (по умолчанию те лежат там как base64, фактически в открытом виде) и отдаёт секрет в под как переменную окружения или файл.

Two hard truths about etcd:

Две жёсткие правды про etcd:

* 🔓 **By default, etcd stores Secrets as base64 (plaintext-equivalent).** Anyone with disk access to an etcd member, an etcd backup, or a snapshot can dump all your secrets.
  * 🔓 **По умолчанию etcd хранит Secrets как base64 (фактически в открытом виде).** Любой, у кого есть доступ к диску узла etcd, к бэкапу или снапшоту, может выгрузить все твои секреты.
* 🛡️ **RBAC protects the API path, not the data at rest.** A node with `etcdctl get /registry/secrets/...` skips the API server entirely.
  * 🛡️ **RBAC защищает путь через API, а не данные на диске.** Узел с `etcdctl get /registry/secrets/...` обходит API-сервер целиком.

> 🤔 **Question:** does your managed control plane (EKS, GKE, AKS) encrypt etcd? **Often yes** — but at the *disk* level (cloud-provider encryption), not the K8s `EncryptionConfiguration` level. They're different threat models.
>
> 🤔 **Вопрос:** шифрует ли твой управляемый control plane (EKS, GKE, AKS) etcd? **Часто да** — но на уровне *диска* (шифрование облачного провайдера), а не на уровне `EncryptionConfiguration` в Kubernetes. Это разные модели угроз.

---

## 📍 Slide 8 – 🔐 Encryption-at-Rest in etcd

Слайд 8 — Шифрование при хранении в etcd

K8s ships an `EncryptionConfiguration` API: the kube-apiserver encrypts Secret payloads **before** writing to etcd.

В Kubernetes есть API `EncryptionConfiguration`: kube-apiserver шифрует содержимое Secret **до** записи в etcd.

```yaml
# /etc/kubernetes/enc/enc-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: ["secrets"]
    providers:
      - kms:                           # 🥇 prefer KMS provider (AWS KMS, GCP KMS, Vault)
          name: aws-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          cachesize: 1000
      - aescbc:                        # 🥈 fallback: local AES-256-CBC
          keys:
            - name: key1
              secret: <base64-32-byte-key>
      - identity: {}                   # ⚠️ identity = no encryption; must be LAST
```

Пояснение к коду: провайдеры проверяются сверху вниз — сначала `kms`, затем `aescbc`, а `identity` (без шифрования) обязан идти последним, иначе новые секреты запишутся в открытом виде.

Activation:

Активация:

* 🛠️ kube-apiserver flag: `--encryption-provider-config=/etc/kubernetes/enc/enc-config.yaml`
  * 🛠️ Флаг kube-apiserver: `--encryption-provider-config=/etc/kubernetes/enc/enc-config.yaml`
* 🔄 **Re-encrypt existing secrets** after enabling: `kubectl get secrets -A -o json | kubectl replace -f -`
  * 🔄 **Перешифровать существующие секреты** после включения: `kubectl get secrets -A -o json | kubectl replace -f -`
* 🔑 **Rotation** = add a new key at top of the list, replace all secrets, then move the old key down or remove it
  * 🔑 **Ротация** — добавить новый ключ в начало списка, перезаписать все секреты, затем опустить старый ключ ниже или удалить его

> ⚠️ **Gotcha:** if `identity` isn't last, secrets stay unencrypted. Order matters.
>
> ⚠️ **Подвох:** если `identity` не последний, секреты остаются незашифрованными. Порядок важен.

---

## 📍 Slide 9 – 🪦 The Vault → OpenBao Story

Слайд 9 — История Vault → OpenBao

```mermaid
timeline
  title HashiCorp Vault → OpenBao
  2015 : Vault 0.1 released by HashiCorp (MPL-2.0)
  2023-Aug-10 : HashiCorp re-licenses Vault under BSL 1.1 (non-compete clause)
  2023-Dec : Linux Foundation announces OpenBao (fork of Vault 1.14, MPL-2.0)
  2024-Mar : OpenBao 2.0 GA — drop-in replacement for Vault
  2025 : Major distros stop shipping Vault, switch to OpenBao
  2026-Feb-04 : OpenBao 2.5.0 — Namespaces (free), horizontal read scalability
```

Схема: хронология — Vault 0.1 от HashiCorp в 2015-м, перелицензирование под BSL 1.1 в августе 2023-го, анонс форка OpenBao под эгидой Linux Foundation в декабре 2023-го, GA версии 2.0 в марте 2024-го, переход основных дистрибутивов на OpenBao в 2025-м и релиз 2.5.0 в феврале 2026-го.

* ⚖️ **BSL 1.1** = source-available but you may not offer Vault as a commercial managed service competing with HashiCorp.
  * ⚖️ **BSL 1.1** — исходники открыты, но нельзя предлагать Vault как коммерческий managed-сервис, конкурирующий с HashiCorp.
* 🍴 The OpenBao fork is governed by the **Linux Foundation**, MPL-2.0 (true open source), wire-compatible with the `vault` CLI.
  * 🍴 Форком OpenBao управляет **Linux Foundation**, лицензия MPL-2.0 (настоящий open source), совместимость с CLI `vault` на уровне протокола.
* 💡 The **paid HashiCorp features** (Namespaces, horizontal read scaling, MFA) are **free in OpenBao 2.5.0**.
  * 💡 **Платные функции HashiCorp** (Namespaces, горизонтальное масштабирование чтения, MFA) **в OpenBao 2.5.0 бесплатны**.

> 🔥 **Why we teach OpenBao, not Vault:** the lab will run on student laptops. BSL says you can do that. But every reusable manifest you build today is going to a production cluster tomorrow — start with the license that doesn't get re-litigated.
>
> 🔥 **Почему мы учим OpenBao, а не Vault:** лаба запускается на ноутбуках студентов, и BSL это разрешает. Но каждый переиспользуемый манифест, который ты соберёшь сегодня, завтра пойдёт в продовый кластер — начинай с лицензии, которую не придётся пересматривать в суде.

---

## 📍 Slide 10 – 🏰 OpenBao Architecture

Слайд 10 — Архитектура OpenBao

```mermaid
flowchart TD
  subgraph Clients["👥 Clients"]
    P[📦 Pods]
    C[💻 vault CLI]
    A[🔌 API / SDK]
  end
  subgraph Server["🏰 OpenBao Server"]
    Auth[🔑 Auth Methods<br/>k8s, AppRole, OIDC]
    Pol[📋 Policies<br/>HCL ACL]
    Eng[🔐 Secret Engines<br/>kv-v2, database, pki, transit]
    Aud[📊 Audit Devices<br/>file / syslog]
  end
  Store[(💾 Raft / Postgres<br/>encrypted)]
  P --> Auth
  C --> Auth
  A --> Auth
  Auth --> Pol
  Pol --> Eng
  Eng --> Store
  Auth --> Aud
```

Схема: клиенты (поды, CLI `vault`, API/SDK) проходят аутентификацию, затем проверяются политики, запрос уходит в нужный secret engine, а данные сохраняются в зашифрованном хранилище Raft или Postgres; все обращения фиксируются в аудите.

**Three concepts you must know:**

**Три понятия, которые нужно знать:**

* 🔐 **Secret Engines** = where secrets live or are generated. `kv-v2` (versioned static), `database` (dynamic creds), `pki` (cert issuance), `transit` (encryption-as-a-service).
  * 🔐 **Secret Engines** — где секреты хранятся или генерируются: `kv-v2` (версионируемые статические), `database` (динамические учётные данные), `pki` (выдача сертификатов), `transit` (шифрование как сервис).
* 🔑 **Auth Methods** = how clients prove who they are. For K8s pods: `kubernetes` (validates the pod's ServiceAccount JWT against the K8s TokenReview API).
  * 🔑 **Auth Methods** — как клиенты доказывают, кто они. Для подов K8s: `kubernetes` (проверяет JWT ServiceAccount пода через TokenReview API).
* 📋 **Policies** = HCL files describing path-level capabilities (`read`, `list`, `create`, `update`, `delete`, `sudo`, `deny`).
  * 📋 **Policies** — HCL-файлы с правами на уровне путей (`read`, `list`, `create`, `update`, `delete`, `sudo`, `deny`).

---

## 📍 Slide 11 – 🛠️ Install OpenBao via Helm

Слайд 11 — Установка OpenBao через Helm

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update

helm install openbao openbao/openbao \
  --namespace openbao --create-namespace \
  --set "server.dev.enabled=true" \         # 🚨 dev mode = unsealed + in-memory, NEVER prod
  --set "injector.enabled=true" \           # 💉 deploy the Agent Sidecar Injector
  --set "server.image.tag=2.5.0"

kubectl get pods -n openbao
# NAME                                    READY   STATUS
# openbao-0                               1/1     Running
# openbao-agent-injector-7f...            1/1     Running
```

Пояснение: команды ставят OpenBao в dev-режиме и сразу включают инжектор сайдкаров; в выводе `kubectl get pods` должны быть поды `openbao-0` и `openbao-agent-injector`.

* 🚨 **Dev mode** auto-initializes, auto-unseals, stores in RAM. Restart = secret loss. **Lab only.**
  * 🚨 **Dev-режим** сам инициализируется, сам распечатывается и хранит данные в RAM. Перезапуск — потеря секретов. **Только для лабы.**
* 🏢 **Prod:** integrated storage (Raft) on 3-5 nodes, auto-unseal via KMS, audit device → log aggregator, TLS everywhere.
  * 🏢 **Прод:** встроенное хранилище (Raft) на 3–5 узлах, авто-распечатывание через KMS, audit device → сборщик логов, TLS везде.

> 🔥 **The `vault` CLI works.** OpenBao kept the API + CLI compatible. `vault status`, `vault kv put`, `vault login` — same commands.
>
> 🔥 **CLI `vault` работает.** OpenBao сохранил совместимость API и CLI. `vault status`, `vault kv put`, `vault login` — те же команды.

---

## 📍 Slide 12 – 🔑 Configure: KV-v2 + Kubernetes Auth + Policy

Слайд 12 — Настройка: KV-v2, Kubernetes auth и политика

```bash
kubectl exec -n openbao -it openbao-0 -- /bin/sh

# 🔐 Enable KV v2 (versioned secrets)
vault secrets enable -path=secret kv-v2

# 📝 Write a secret
vault kv put secret/lab11/db \
  username=app password='S3cure!2026'

# 🔑 Enable Kubernetes auth (validates pod ServiceAccount JWT)
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# 📋 Policy: read-only on this exact path
vault policy write lab11-read - <<EOF
path "secret/data/lab11/db" {
  capabilities = ["read"]
}
EOF

# 🎯 Bind the policy to a ServiceAccount + namespace
vault write auth/kubernetes/role/lab11 \
  bound_service_account_names=lab11-sa \
  bound_service_account_namespaces=lab11 \
  policies=lab11-read \
  ttl=1h
```

Пояснение: по шагам — включаем KV-v2 по пути `secret`, кладём пару логин/пароль, включаем Kubernetes auth, пишем политику только на чтение `secret/data/lab11/db` и создаём роль `lab11`, привязанную к ServiceAccount `lab11-sa` в пространстве имён `lab11` на 1 час.

* 🎯 The **role** ties three things together: which SA can authenticate, which policy gets attached, and how long the resulting token lasts.
  * 🎯 **Роль** связывает три вещи: какой ServiceAccount может аутентифицироваться, какая политика к нему прикрепляется и сколько живёт полученный токен.

---

## 📍 Slide 13 – 💉 Vault Agent Injector: The Sidecar Pattern

Слайд 13 — Vault Agent Injector: паттерн сайдкара

```mermaid
sequenceDiagram
  participant API as kube-apiserver
  participant Inj as Agent Injector<br/>(MutatingWebhook)
  participant Pod
  participant Bao as OpenBao
  Note over Pod: Pod created with vault.hashicorp.com/* annotations
  API->>Inj: AdmissionReview
  Inj->>API: Patched podspec (+initContainer +sidecar)
  API->>Pod: schedule
  Pod->>Bao: ServiceAccount JWT
  Bao->>Pod: Vault token (matches role policy)
  Pod->>Bao: read secret/lab11/db
  Bao->>Pod: payload
  Pod->>Pod: write /vault/secrets/db (shared emptyDir)
  Note over Pod: app container reads file
```

Схема: при создании пода с аннотациями `vault.hashicorp.com/*` webhook-инжектор правит podspec, добавляя init-контейнер и сайдкар; под аутентифицируется в OpenBao по JWT своего ServiceAccount, получает токен по политике роли, читает секрет и записывает его в файл на общем `emptyDir`, откуда приложение его и читает.

* 🪝 The injector is a **MutatingAdmissionWebhook** — it edits the podspec at admission time, transparent to the chart author beyond annotations.
  * 🪝 Инжектор — это **MutatingAdmissionWebhook**: он правит podspec в момент admission, и для автора чарта это прозрачно, кроме аннотаций.
* 🔄 An **init container** fetches the secret before the app starts (so the file exists at app boot). A **sidecar** keeps renewing the token + refreshing the file.
  * 🔄 **Init-контейнер** забирает секрет до старта приложения (чтобы файл был уже на момент запуска). **Сайдкар** продлевает токен и обновляет файл.
* 📁 Secrets land in `/vault/secrets/<name>` on a shared `emptyDir` volume.
  * 📁 Секреты попадают в `/vault/secrets/<name>` на общий том `emptyDir`.

---

## 📍 Slide 14 – 🏷️ Vault Agent Annotations

Слайд 14 — Аннотации Vault Agent

```yaml
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "lab11"
        # 🎯 Plain key=value file at /vault/secrets/db
        vault.hashicorp.com/agent-inject-secret-db: "secret/data/lab11/db"
        # 🎨 Custom template — render as .env
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "secret/data/lab11/db" -}}
          DB_USER={{ .Data.data.username }}
          DB_PASS={{ .Data.data.password }}
          {{- end -}}
        # 🔄 Re-render every 30s; signal SIGHUP to PID 1
        vault.hashicorp.com/agent-inject-default-template: "json"
        vault.hashicorp.com/agent-inject-command-db: "kill -HUP 1"
    spec:
      serviceAccountName: lab11-sa
      containers: [...]
```

Пояснение: аннотации `agent-inject` и `role` включают инжектор и задают роль; `agent-inject-secret-db` указывает исходный секрет, `agent-inject-template-db` — шаблон вывода в формате `.env`, а `agent-inject-command-db` посылает `SIGHUP` процессу 1 после обновления.

* 🎨 `agent-inject-template-*` lets you render the secret in **any format** — `.env`, JSON, YAML, HCL, a full app config file.
  * 🎨 `agent-inject-template-*` позволяет отрендерить секрет в **любом формате** — `.env`, JSON, YAML, HCL, целый конфиг приложения.
* 🔁 `agent-inject-command-*` runs an in-pod command after re-rendering (signal the app to reload config).
  * 🔁 `agent-inject-command-*` выполняет команду внутри пода после перерендеринга (сигнал приложению перечитать конфиг).

---

## 📍 Slide 15 – 🤖 External Secrets Operator (ESO): The Other Pattern

Слайд 15 — External Secrets Operator (ESO): другой паттерн

ESO **syncs** secrets from an external store → a native K8s `Secret`. No sidecar. Apps read a normal `Secret` and don't know ESO exists.

ESO **синхронизирует** секреты из внешнего хранилища → в нативный K8s `Secret`. Никакого сайдкара. Приложение читает обычный `Secret` и не знает, что ESO существует.

```mermaid
flowchart LR
  Bao[🏰 OpenBao] -->|👀 watch| ESO[🤖 ESO controller]
  ESO -->|✍️ create/update| Sec[(🔐 K8s Secret)]
  Sec --> App[📦 App Pod<br/>envFrom: secretRef]
```

Схема: контроллер ESO следит за OpenBao, создаёт и обновляет K8s `Secret`, а под приложения читает его через `envFrom: secretRef`.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata: {name: openbao, namespace: lab11}
spec:
  provider:
    vault:                                   # ✅ ESO's vault provider works with OpenBao
      server: "http://openbao.openbao:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "lab11"
          serviceAccountRef: {name: lab11-sa}
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: db, namespace: lab11}
spec:
  refreshInterval: 1h                        # ⏱️ controller polls every hour
  secretStoreRef: {name: openbao, kind: SecretStore}
  target: {name: db-creds}                   # ⬅️ produces a K8s Secret with this name
  data:
    - secretKey: password
      remoteRef: {key: lab11/db, property: password}
```

Пояснение: `SecretStore` описывает подключение к OpenBao (провайдер `vault`, KV версии 2, аутентификация по Kubernetes с ролью `lab11`), а `ExternalSecret` велит ESO раз в час (`refreshInterval`) брать из `secret/lab11/db` поле `password` и складывать его в `Secret` с именем `db-creds`.

---

## 📍 Slide 16 – 💉 Agent Injector vs ESO vs CSI: Pick One

Слайд 16 — Agent Injector vs ESO vs CSI: выбери одно

| Concern | Agent Injector | ESO | Secrets Store CSI |
|---------|----------------|-----|-------------------|
| 📦 Secret lives in K8s `Secret`? | ❌ file only | ✅ yes | ⚠️ optional sync |
| 🔁 Auto-refresh on rotation | ✅ live (template + signal) | ⚠️ poll interval | ⚠️ on pod restart |
| 🧊 Pod resource cost | ❌ sidecar per pod | ✅ one controller | ✅ daemonset per node |
| 🔑 Supported auth methods | ✅ all Vault methods | ✅ all | ⚠️ k8s only |
| 🎯 Best when | dynamic creds, short TTL, signal-driven reloads | most pods want `envFrom`, polling is fine | strict no-K8s-Secret rules, file mounts |
| 🚦 Complexity | 🟡 medium | 🟢 low | 🟡 medium |

| Критерий | Agent Injector | ESO | Secrets Store CSI |
|---------|----------------|-----|-------------------|
| 📦 Секрет живёт в K8s `Secret`? | ❌ только файл | ✅ да | ⚠️ опциональная синхронизация |
| 🔁 Автообновление при ротации | ✅ в реальном времени (шаблон и сигнал) | ⚠️ по интервалу опроса | ⚠️ при перезапуске пода |
| 🧊 Затраты ресурсов пода | ❌ сайдкар в каждом поде | ✅ один контроллер | ✅ daemonset на каждом узле |
| 🔑 Поддерживаемые методы аутентификации | ✅ все методы Vault | ✅ все | ⚠️ только k8s |
| 🎯 Когда лучший выбор | динамические учётные данные, короткий TTL, перезагрузка по сигналу | большинству подов нужен `envFrom`, опроса достаточно | строгий запрет на K8s Secret, монтирование файлов |
| 🚦 Сложность | 🟡 средняя | 🟢 низкая | 🟡 средняя |

> 🔥 **Most teams in 2026 pick ESO** as the default and add Agent Injector only for the pods that need dynamic database creds or sub-minute rotation.
>
> 🔥 **Большинство команд в 2026-м выбирают ESO** по умолчанию, а Agent Injector добавляют только для подов, которым нужны динамические учётные данные БД или ротация чаще раза в минуту.

---

## 📍 Slide 17 – 🌳 Alternatives: SOPS and Sealed Secrets

Слайд 17 — Альтернативы: SOPS и Sealed Secrets

Not every team can run a Vault/OpenBao cluster. Two GitOps-friendly alternatives:

Не каждая команда может позволить себе кластер Vault/OpenBao. Две GitOps-альтернативы:

* 🔏 **Mozilla SOPS** — encrypt YAML/JSON **file-level** with AWS KMS / GCP KMS / age / PGP. Encrypted file is safe to commit. The CI/CD pipeline (or `helm-secrets` plugin) decrypts at apply time.
  ```bash
  sops --encrypt --age $AGE_RECIPIENT secrets.yaml > secrets.enc.yaml
  # commit secrets.enc.yaml, *.gitignore secrets.yaml
  ```
  * 🔏 **Mozilla SOPS** — шифрует YAML/JSON **на уровне файла** через AWS KMS / GCP KMS / age / PGP. Зашифрованный файл можно безопасно коммитить. CI/CD-пайплайн (или плагин `helm-secrets`) расшифровывает его во время применения.
* 🔒 **Sealed Secrets** (Bitnami) — a CRD + controller in-cluster. Encrypt with the controller's public key locally; only that one cluster can decrypt. Encrypted resource is safe to commit and apply.
  ```bash
  kubeseal --controller-namespace=kube-system < secret.yaml > sealed.yaml
  ```
  * 🔒 **Sealed Secrets** (Bitnami) — CRD и контроллер в кластере. Шифруешь локально публичным ключом контроллера; расшифровать сможет только этот кластер. Зашифрованный ресурс безопасно коммитить и применять.

| | SOPS | Sealed Secrets | OpenBao + ESO |
|---|------|----------------|---------------|
| 🔄 Rotation | ❌ manual re-encrypt | ❌ manual re-encrypt | ✅ central |
| 🔍 Audit log | ❌ git history | ❌ git history | ✅ Vault audit |
| 🌍 Multi-cluster sharing | ✅ | ❌ per-cluster key | ✅ |
| 🎚️ Complexity | 🟢 low | 🟢 low | 🟡 medium |

| | SOPS | Sealed Secrets | OpenBao + ESO |
|---|------|----------------|---------------|
| 🔄 Ротация | ❌ вручную перешифровывать | ❌ вручную перешифровывать | ✅ централизованно |
| 🔍 Журнал аудита | ❌ история git | ❌ история git | ✅ аудит Vault |
| 🌍 Обмен между кластерами | ✅ | ❌ ключ своего кластера | ✅ |
| 🎚️ Сложность | 🟢 низкая | 🟢 низкая | 🟡 средняя |

---

## 📍 Slide 18 – 🪞 OpenBao 2.5.0: What's New (Feb 4 2026)

Слайд 18 — OpenBao 2.5.0: что нового (4 февраля 2026)

* 🏷️ **Namespaces** are now free — multi-tenant secret isolation that used to be a HashiCorp Enterprise paywall feature. Each namespace gets its own policies/auth/mounts.
  * 🏷️ **Namespaces** теперь бесплатны — изоляция секретов между арендаторами, которая раньше была платной функцией HashiCorp Enterprise. У каждого namespace свои политики, методы аутентификации и точки монтирования.
* 🚀 **Horizontal read scalability** — standby Raft nodes can now answer **read** requests locally without forwarding to the leader. Read-heavy workloads (think 10K pods all polling a config secret) suddenly become cheap.
  * 🚀 **Горизонтальная масштабируемость чтения** — standby-узлы Raft теперь отвечают на запросы **чтения** локально, не пересылая их лидеру. Нагрузки с преобладанием чтения (например, 10 тыс. подов, опрашивающих конфиг-секрет) внезапно становятся дешёвыми.
* 🔒 **`disable_unauthed_rekey_endpoints: true` by default** — closes an unauth attack surface (CVE-class concern in older Vault).
  * 🔒 **`disable_unauthed_rekey_endpoints: true` по умолчанию** — закрывает поверхность атаки без аутентификации (проблема класса CVE в старых версиях Vault).
* 🛠️ Auto-unseal hardening for managed keys (AWS KMS / GCP KMS / Azure Key Vault).
  * 🛠️ Усиление авто-распечатывания для управляемых ключей (AWS KMS / GCP KMS / Azure Key Vault).
* 🧹 Identity-group cleanup on unseal — fixes a long-standing Vault bug where corrupted entries blocked startup.
  * 🧹 Очистка групп идентичности при распечатывании — исправляет давнюю ошибку Vault, из-за которой повреждённые записи блокировали запуск.

> 🔥 **Practical impact:** the `vault` CLI commands you write today against OpenBao will run unchanged on production setups at any scale. Old Vault tutorials remain ~95% accurate.
>
> 🔥 **Практический эффект:** команды CLI `vault`, которые ты пишешь сегодня против OpenBao, без изменений заработают в проде любого масштаба. Старые туториалы по Vault остаются верными примерно на 95%.

---

## 📍 Slide 19 – 🚫 Anti-Patterns

Слайд 19 — Антипаттерны

1. ❌ **`echo $PASSWORD | base64` and committing the result** — base64 is encoding, not encryption. Git history is forever.
   * ❌ **`echo $PASSWORD | base64` и коммит результата** — base64 это кодирование, а не шифрование. История git вечна.
2. ❌ **Plain `Secret` with no etcd encryption** — anyone with an etcd snapshot owns your prod.
   * ❌ **Обычный `Secret` без шифрования etcd** — у кого есть снапшот etcd, тот владеет твоим продом.
3. ❌ **One Vault token per app, never rotated** — defeats the entire point of a secret manager.
   * ❌ **Один токен Vault на приложение, без ротации** — убивает весь смысл менеджера секретов.
4. ❌ **Vault dev mode in production** — RAM storage, single key, auto-unseal. Process restart = total secret loss.
   * ❌ **Dev-режим Vault в проде** — хранение в RAM, один ключ, авто-распечатывание. Перезапуск процесса = полная потеря секретов.
5. ❌ **Long-lived ServiceAccount tokens mounted by default** — set `automountServiceAccountToken: false` on SAs that don't need Vault auth.
   * ❌ **Долгоживущие токены ServiceAccount, смонтированные по умолчанию** — ставь `automountServiceAccountToken: false` для SA, которым не нужна аутентификация в Vault.
6. ❌ **`vault.hashicorp.com/role: admin` on every pod** — least privilege; one role per app per environment.
   * ❌ **`vault.hashicorp.com/role: admin` на каждом поде** — принцип наименьших привилегий: своя роль на каждое приложение в каждом окружении.
7. ❌ **No secret rotation cadence** — pick a number (30/60/90 days), automate it, audit deviations.
   * ❌ **Отсутствие графика ротации секретов** — выбери число (30/60/90 дней), автоматизируй его и проверяй отклонения.
8. ❌ **Putting real values in `values.yaml`** — that file lands in git via your Helm chart. Use ESO or Agent Injector, never `--set` from a CI step that logs.
   * ❌ **Настоящие значения в `values.yaml`** — этот файл попадает в git вместе с Helm-чартом. Используй ESO или Agent Injector и никогда не передавай `--set` из шага CI, который пишет логи.

---

## 📍 Slide 20 – 🧪 Detection: How Leaks Get Found

Слайд 20 — Обнаружение: как находят утечки

The good news: secret-scanning is cheap, fast, and CI-friendly.

Хорошая новость: сканирование секретов дешёвое, быстрое и хорошо ложится на CI.

* 🔍 **gitleaks**, **trufflehog**, **detect-secrets** — pre-commit hooks + CI gates. Find AWS keys, GCP service accounts, PEM keys, GH tokens.
  * 🔍 **gitleaks**, **trufflehog**, **detect-secrets** — pre-commit-хуки и проверки в CI. Находят ключи AWS, сервисные аккаунты GCP, PEM-ключи, токены GitHub.
* 🐙 **GitHub Secret Scanning** — built into github.com; partner integrations auto-revoke leaked tokens for AWS, GCP, Slack, Stripe, …
  * 🐙 **GitHub Secret Scanning** — встроен в github.com; партнёрские интеграции автоматически отзывают утёкшие токены AWS, GCP, Slack, Stripe и других сервисов.
* 📊 **OWASP Top 10 (A07: Identification and Authentication Failures)** — explicitly calls out credential management as a top-tier risk.
  * 📊 **OWASP Top 10 (A07: Identification and Authentication Failures)** — прямо называет управление учётными данными риском верхнего уровня.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks: [{id: gitleaks}]
```

Пояснение: этот `.pre-commit-config.yaml` подключает хук `gitleaks` версии v8.21.2, который запускается при каждом коммите.

> 🔥 **Run gitleaks on every PR.** It catches the human moments. Tools beat policy.
>
> 🔥 **Запускай gitleaks на каждом PR.** Он ловит человеческий фактор. Инструменты надёжнее правил.

---

## 📍 Slide 21 – 🗺️ Decision Tree

Слайд 21 — Дерево решений

```mermaid
flowchart TD
  Start[🤔 I have a secret to manage] --> Q1{Single cluster?<br/>Small team?<br/>No compliance ask?}
  Q1 -->|Yes| KMS[☸️ K8s Secret<br/>+ etcd KMS provider<br/>+ SOPS for git-tracked]
  Q1 -->|No| Q2{Workload outside<br/>K8s too?}
  Q2 -->|Yes| Bao[🏰 OpenBao centrally<br/>+ ESO inside clusters]
  Q2 -->|No| Q3{Need dynamic creds<br/>or sub-minute rotation?}
  Q3 -->|Yes| Agent[💉 OpenBao + Agent Injector]
  Q3 -->|No| ESO[🤖 OpenBao + ESO]
```

Схема: если кластер один, команда небольшая и нет требований комплаенса — хватит K8s Secret с KMS-провайдером etcd и SOPS для файлов в git; если есть нагрузки вне K8s — OpenBao централизованно и ESO внутри кластеров; если нужны динамические учётные данные или ротация чаще раза в минуту — Agent Injector.

* 🟢 **Start small:** K8s Secret + etcd encryption gets you 80% of the way.
  * 🟢 **Начинай с малого:** K8s Secret и шифрование etcd дают 80% результата.
* 🟡 **Add OpenBao + ESO** when you have ≥2 clusters or non-K8s consumers.
  * 🟡 **Добавляй OpenBao + ESO**, когда кластеров ≥2 или есть потребители вне K8s.
* 🔴 **Add Agent Injector + dynamic creds** when audit/compliance demands it.
  * 🔴 **Добавляй Agent Injector и динамические учётные данные**, когда этого требуют аудит и комплаенс.

---

## 📍 Slide 22 – 🎯 Key Takeaways

Слайд 22 — Главное из лекции

1. 🔓 **K8s `Secret` is base64, not encryption.** Confidentiality comes from RBAC + etcd-at-rest, not the resource itself.
   * 🔓 **K8s `Secret` — это base64, а не шифрование.** Конфиденциальность дают RBAC и шифрование etcd, а не сам ресурс.
2. 🔐 **Turn on `EncryptionConfiguration`** with a KMS provider; `identity` always last.
   * 🔐 **Включи `EncryptionConfiguration`** с провайдером KMS; `identity` всегда последний.
3. 🪦 **HashiCorp Vault went BSL in Aug 2023.** The community moved to **OpenBao** (LF / MPL-2.0). The `vault` CLI still works.
   * 🪦 **HashiCorp Vault перешёл на BSL в августе 2023-го.** Сообщество переехало на **OpenBao** (Linux Foundation / MPL-2.0). CLI `vault` по-прежнему работает.
4. 🏰 **OpenBao 2.5.0** gives you free Namespaces + horizontal read scaling.
   * 🏰 **OpenBao 2.5.0** даёт бесплатные Namespaces и горизонтальное масштабирование чтения.
5. 💉 **Two injection patterns:** Agent Injector (sidecar, dynamic, signal-driven) and ESO (controller-synced K8s Secret). Pick by workload, not by hype.
   * 💉 **Два паттерна внедрения:** Agent Injector (сайдкар, динамика, сигналы) и ESO (синхронизированный контроллером K8s Secret). Выбирай по нагрузке, а не по хайпу.
6. 🌳 **SOPS / Sealed Secrets** are valid GitOps-friendly fallbacks when you can't run a Vault cluster.
   * 🌳 **SOPS / Sealed Secrets** — рабочие GitOps-альтернативы, если нет возможности держать кластер Vault.
7. 🔍 **Scan every commit** with gitleaks/trufflehog. Detection is cheaper than rotation.
   * 🔍 **Сканируй каждый коммит** через gitleaks/trufflehog. Обнаружение дешевле ротации.

> 💬 *"Security is not a product, but a process."* — Bruce Schneier
>
> 💬 *«Безопасность — не продукт, а процесс»*. — Брюс Шнайер

---

## 📍 Slide 23 – 🚀 What Comes Next

Слайд 23 — Что дальше

**📚 Next lecture: *ConfigMaps & Persistent Volumes*** — non-sensitive configuration, mounting strategies, and how stateful apps survive pod restarts.

**📚 Следующая лекция: *ConfigMaps и Persistent Volumes*** — несекретная конфигурация, стратегии монтирования и то, как stateful-приложения переживают перезапуск подов.

* 📁 `ConfigMap` vs `Secret` — when each is the right answer
  * 📁 `ConfigMap` vs `Secret` — когда что уместнее
* 💾 PersistentVolume / PersistentVolumeClaim / StorageClass — dynamic provisioning
  * 💾 PersistentVolume / PersistentVolumeClaim / StorageClass — динамическое выделение
* 🔧 Mount strategies: subPath, projected volumes, immutable configs
  * 🔧 Стратегии монтирования: subPath, projected volumes, неизменяемые конфиги
* 🗂️ StatefulSets vs Deployments — when ordering matters
  * 🗂️ StatefulSets vs Deployments — когда важен порядок

**🔬 Lab 11 deliverables:**

**🔬 Что нужно сдать по лабе 11:**

* Create an `app-credentials` Secret with `kubectl create secret`, view + decode it
  * Создать `Secret` с именем `app-credentials` через `kubectl create secret`, посмотреть и декодировать его
* Add `templates/secrets.yaml` to your Lab 10 chart, inject via `envFrom`
  * Добавить `templates/secrets.yaml` в свой чарт из лабы 10 и внедрить секрет через `envFrom`
* Install **OpenBao** in dev mode via the OpenBao Helm chart
  * Установить **OpenBao** в dev-режиме через Helm-чарт OpenBao
* Enable KV-v2 + Kubernetes auth + a read policy + a role bound to your SA
  * Включить KV-v2, Kubernetes auth, политику на чтение и роль, привязанную к твоему SA
* Add Vault Agent annotations to your Deployment; verify `/vault/secrets/...` appears in the pod
  * Добавить аннотации Vault Agent в свой Deployment и убедиться, что внутри пода появился `/vault/secrets/...`
* Document everything in `k8s/SECRETS.md`
  * Описать всё в `k8s/SECRETS.md`
* **Bonus (+2.5 pts):** custom `agent-inject-template-*` rendering a `.env`, named template in `_helpers.tpl`, document the refresh mechanism
  * **Бонус (+2,5 балла):** свой `agent-inject-template-*`, рендерящий `.env`, именованный шаблон в `_helpers.tpl` и описание механизма обновления

```mermaid
flowchart LR
  Lab10[📦 Lab 10: Helm chart] --> Lab11[🔐 Lab 11: Secrets + OpenBao]
  Lab11 --> Lab12[📁 Lab 12: ConfigMaps + PV]
  Lab12 --> Lab13[🔄 Lab 13: ArgoCD]
```

Схема: чарт из лабы 10 превращается в лабу 11 с секретами и OpenBao, затем в лабу 12 с ConfigMaps и PersistentVolume, а потом в лабу 13 с ArgoCD.

> 🌊 You've stopped putting passwords in git. Now let's configure everything else.
>
> 🌊 Ты перестал складывать пароли в git. Теперь настроим всё остальное.

---

## 📚 Resources

Материалы

* 📕 *Container Security* — Liz Rice (O'Reilly, 2020) — chapter on Secrets and threat model
  * 📕 *Container Security* — Лиз Райс (O'Reilly, 2020) — глава про Secrets и модель угроз
* 📕 *Kubernetes Security and Observability* — Brendan Creane & Amit Gupta (O'Reilly, 2021)
  * 📕 *Kubernetes Security and Observability* — Брендан Крейн и Амит Гупта (O'Reilly, 2021)
* 🌐 [openbao.org/docs](https://openbao.org/docs/) — OpenBao official docs
  * 🌐 [openbao.org/docs](https://openbao.org/docs/) — официальная документация OpenBao
* 🌐 [OpenBao 2.5.0 release notes](https://openbao.org/community/release-notes/2-5-0/) — Feb 4 2026
  * 🌐 [Release notes OpenBao 2.5.0](https://openbao.org/community/release-notes/2-5-0/) — 4 февраля 2026
* 🌐 [external-secrets.io](https://external-secrets.io/) — ESO docs + provider list
  * 🌐 [external-secrets.io](https://external-secrets.io/) — документация ESO и список провайдеров
* 🌐 [kubernetes.io/docs/concepts/configuration/secret](https://kubernetes.io/docs/concepts/configuration/secret/) — K8s Secrets reference
  * 🌐 [kubernetes.io/docs/concepts/configuration/secret](https://kubernetes.io/docs/concepts/configuration/secret/) — справочник по K8s Secrets
* 🌐 [kubernetes.io/docs/tasks/administer-cluster/encrypt-data](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) — encryption-at-rest setup
  * 🌐 [kubernetes.io/docs/tasks/administer-cluster/encrypt-data](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) — настройка шифрования при хранении
* 🌐 [getsops.io](https://getsops.io/) — SOPS docs
  * 🌐 [getsops.io](https://getsops.io/) — документация SOPS
* 🌐 [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) — Sealed Secrets
  * 🌐 [github.com/bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) — Sealed Secrets
* 🎥 *Kubernetes Secrets Are Not Secret* — KubeCon talk by Liz Rice (covers etcd attack surface)
  * 🎥 *Kubernetes Secrets Are Not Secret* — доклад Лиз Райс на KubeCon (разбирает поверхность атаки на etcd)

**🎓 Quiz:** Post-lecture quiz feeds the weeks 10-12 leaderboard window.

**🎓 Квиз:** квиз после лекции идёт в зачёт лидерборда недель 10–12.
