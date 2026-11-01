# Lab 4 — Infrastructure as Code (Terraform & Pulumi)

Лабораторная 4 — Инфраструктура как код (Terraform и Pulumi)

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Infrastructure%20as%20Code-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Terraform%20%7C%20Pulumi-informational)

> **Goal:** Provision the *same* small VM twice — once in **Terraform** HCL, once in **Pulumi** Python — and feel the trade-offs in your hands.
> **Deliverable:** A PR from `lab04` with `terraform/`, `pulumi/`, `docs/LAB04.md`, and (bonus) `.github/workflows/terraform-ci.yml`.
>
> **Цель:** поднять *одну и ту же* небольшую VM дважды — сначала на HCL в **Terraform**, затем на Python в **Pulumi** — и на практике прочувствовать компромиссы.
> **Что сдаём:** PR из ветки `lab04` с `terraform/`, `pulumi/`, `docs/LAB04.md` и (бонус) `.github/workflows/terraform-ci.yml`.

---

## Overview

Обзор

In this lab you will practice:

В этой лабе ты отработаешь:

- The **`init → plan → apply → destroy`** lifecycle (and reading the plan like a diff)
  * Жизненный цикл **`init → plan → apply → destroy`** (и чтение плана как diff)
- Writing the four HCL primitives — `provider`, `resource`, `variable`, `output` — and one `data` source
  * Написание четырёх базовых блоков HCL — `provider`, `resource`, `variable`, `output` — и одного источника `data`
- Recreating the same infrastructure in Pulumi Python (real loops, real types, real imports)
  * Воссоздание той же инфраструктуры на Pulumi Python (настоящие циклы, типы и импорты)
- **State management** — what's in `terraform.tfstate`, why it's a secret, why locking matters
  * **Управление state** — что лежит в `terraform.tfstate`, почему это секрет и почему важна блокировка

> ⚠️ **Scope:** one cloud VM (or one docker-provider stand-in — see note below). No Kubernetes, no app deploy yet. Lab 5 will SSH into whatever you build here.
>
> ⚠️ **Объём:** одна облачная VM (или одна замена на docker-provider — см. примечание ниже). Никакого Kubernetes и деплоя приложения пока нет. Лаба 5 подключится по SSH к тому, что ты здесь построишь.

**Cloud or docker-provider?** Pick a free-tier cloud (recommended — that's the real skill), but if you can't open a card-protected account, the `kreuzwerker/docker` provider runs the *exact same lifecycle* locally against a container. **Lab 5 cross-ref for docker users:** the container has no sshd; Lab 5 will use `ansible_connection: community.docker.docker` (or run Ansible with `connection: local` against `docker exec`) instead of SSH — same roles, different connection plugin. Either path scores full marks; document which you picked.

**Облако или docker-provider?** Выбирай облако с бесплатным тарифом (рекомендую — это и есть настоящий навык), но если не можешь открыть аккаунт с привязкой карты, провайдер `kreuzwerker/docker` прогоняет *точно такой же жизненный цикл* локально на контейнере. **Перекрёстная ссылка на лабу 5 для тех, кто выбрал docker:** в контейнере нет sshd, поэтому лаба 5 будет использовать `ansible_connection: community.docker.docker` (или запускать Ansible с `connection: local` и `docker exec`) вместо SSH — те же роли, другой connection plugin. Любой путь даёт полный балл; задокументируй, какой выбрал.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно остаться от предыдущих лаб:**

- Lab 1: `app_python/` (and maybe `app_go/`) — the service you'll run on the VM
  * Лаба 1: `app_python/` (а может, и `app_go/`) — сервис, который ты запустишь на VM
- Lab 2: a multi-stage Docker image pushed to GHCR
  * Лаба 2: многоступенчатый Docker-образ, запушенный в GHCR
- Lab 3: GitHub Actions building/pushing on every PR
  * Лаба 3: GitHub Actions собирает и пушит образ на каждый PR

**This lab adds:**

**Эта лаба добавляет:**

- `terraform/` — HCL you wrote yourself, provisioning **one VM** (or one container)
  * `terraform/` — HCL, который ты написал сам и который создаёт **одну VM** (или один контейнер)
- `pulumi/` — Python that recreates the *same* infrastructure
  * `pulumi/` — Python, который воссоздаёт *ту же* инфраструктуру
- `docs/LAB04.md` — your run notes + the Terraform-vs-Pulumi comparison
  * `docs/LAB04.md` — твои заметки по прогону плюс сравнение Terraform и Pulumi

By Lab 5 you SSH into this host and Ansible-configure it. By Lab 9 you replace it with k3d, but the IaC mindset is the same.

К лабе 5 ты зайдёшь на этот хост по SSH и настроишь его через Ansible. К лабе 9 ты заменишь его на k3d, но образ мышления IaC останется прежним.

---

## Tech Stack & Versions

Технологический стек и версии

| Tool | Version | Notes |
|------|---------|-------|
| **Terraform** | **1.15.3** | BSL-1.1 since **2023-08-10**. Lock at `>= 1.15.0`. |
| **OpenTofu** | **1.12.0** | MPL-2.0, Linux Foundation. 1:1 drop-in — `tofu` works for every command below. |
| **Pulumi** | **3.243.0** | Apache-2.0 throughout. |
| **AWS provider** | `~> 5.0` | Most-documented cloud path. |
| **GCP provider** | `~> 6.0` | Alternative. |
| **Yandex provider** | `~> 0.115` | Russia-friendly, no card initially. |
| **Docker provider** | `kreuzwerker/docker ~> 3.0` | Free local stand-in if no cloud account. |

| Инструмент | Версия | Примечания |
|------|---------|-------|
| **Terraform** | **1.15.3** | BSL-1.1 с **2023-08-10**. Зафиксируй на `>= 1.15.0`. |
| **OpenTofu** | **1.12.0** | MPL-2.0, Linux Foundation. Полностью совместим — `tofu` работает для всех команд ниже. |
| **Pulumi** | **3.243.0** | Полностью под Apache-2.0. |
| **AWS provider** | `~> 5.0` | Самый документированный облачный путь. |
| **GCP provider** | `~> 6.0` | Альтернатива. |
| **Yandex provider** | `~> 0.115` | Подходит для России, карта на старте не нужна. |
| **Docker provider** | `kreuzwerker/docker ~> 3.0` | Бесплатная локальная замена, если нет облачного аккаунта. |

> **OpenTofu note:** the August 10 2023 BSL re-licensing made OpenTofu necessary; today's 1.12.0 runs identical HCL against the same registry. The grader accepts either CLI.
>
> **Про OpenTofu:** перелицензирование под BSL 10 августа 2023 года сделало OpenTofu необходимым; сегодняшняя версия 1.12.0 выполняет идентичный HCL в том же registry. Проверяющий принимает любой CLI.

---

## Setup

Подготовка

```bash
terraform -version   # or: tofu -version
pulumi version

# Generate an SSH keypair for the VM if you don't have one
ssh-keygen -t ed25519 -f ~/.ssh/lab04 -C "lab04@$(hostname)"
chmod 600 ~/.ssh/lab04
cat  ~/.ssh/lab04.pub          # this is what your IaC will inject
```

Пояснение: первая команда проверяет версию Terraform (или OpenTofu), вторая — версию Pulumi; затем генерируется отдельная пара SSH-ключей для лабы, выставляются права на приватный ключ и печатается публичная часть — именно её IaC вставит в VM.

Pick **one** target and stick with it for both tools:

Выбери **одну** цель и придерживайся её для обоих инструментов:

| Target | Smallest free unit | Provider |
|--------|---------------------|----------|
| **AWS** | `t3.micro` (12 mo free) | `hashicorp/aws` |
| **GCP** | `e2-micro` (always-free zone) | `hashicorp/google` |
| **Yandex** | 2 vCPU @ 20%, 1 GB | `yandex-cloud/yandex` |
| **DigitalOcean** | `$200` student credit | `digitalocean/digitalocean` |
| **Docker (local)** | a container | `kreuzwerker/docker` |

| Цель | Минимальный бесплатный размер | Провайдер |
|--------|---------------------|----------|
| **AWS** | `t3.micro` (12 мес. бесплатно) | `hashicorp/aws` |
| **GCP** | `e2-micro` (зона always-free) | `hashicorp/google` |
| **Yandex** | 2 vCPU @ 20%, 1 ГБ | `yandex-cloud/yandex` |
| **DigitalOcean** | `$200` студенческого кредита | `digitalocean/digitalocean` |
| **Docker (локально)** | контейнер | `kreuzwerker/docker` |

> **No credentials in code.** Use env vars (`AWS_ACCESS_KEY_ID`, `GOOGLE_APPLICATION_CREDENTIALS`, `YC_TOKEN`, …) or a shared profile. Never inline them in `provider {}`.
>
> **Никаких учётных данных в коде.** Используй переменные окружения (`AWS_ACCESS_KEY_ID`, `GOOGLE_APPLICATION_CREDENTIALS`, `YC_TOKEN`, …) или общий профиль. Никогда не вписывай их внутрь `provider {}`.

Directory layout (you'll fill the files yourself):

Структура каталогов (файлы будешь заполнять сам):

```
terraform/
├── main.tf          # provider + resources
├── variables.tf     # inputs
├── outputs.tf       # public IP, ssh command
├── terraform.tfvars # values — GITIGNORED
├── .gitignore
└── README.md

pulumi/
├── __main__.py
├── Pulumi.yaml
├── Pulumi.dev.yaml  # GITIGNORED if it holds secrets
├── requirements.txt
└── .gitignore

docs/LAB04.md
```

Пояснение: в `terraform/` лежат `main.tf` (провайдер и ресурсы), `variables.tf` (входные параметры), `outputs.tf` (публичный IP и команда SSH), `terraform.tfvars` с реальными значениями (он в `.gitignore`) и `README.md`; в `pulumi/` — точка входа `__main__.py`, описание проекта `Pulumi.yaml`, конфигурация стека и `requirements.txt`.

---

## Task 1 — Terraform VM (4 pts)

Задание 1 — VM на Terraform (4 балла)

**Goal:** a reachable VM, written in your own HCL, with clean state hygiene.

**Цель:** доступная VM, написанная на твоём собственном HCL, с чистым управлением state.

### 1.1 — Write the HCL building blocks

1.1 — Пишем строительные блоки HCL

Lecture 4 told you HCL has the six primitives you'll meet today: `terraform {}`, `provider`, `variable`, `data`, `resource`, `output`. This task is where you write them.

В лекции 4 говорилось, что в HCL есть шесть примитивов, с которыми ты сегодня встретишься: `terraform {}`, `provider`, `variable`, `data`, `resource`, `output`. В этом задании ты их и напишешь.

`YOUR TASK`: produce `terraform/main.tf` covering **all** of the following. Don't pull a copy off Stack Overflow — that's how secrets end up in tfstate.

`YOUR TASK`: подготовь `terraform/main.tf`, покрывающий **всё** перечисленное ниже. Не копируй готовое со Stack Overflow — именно так секреты и попадают в tfstate.

Required HCL **shape** (fill every `___` and every `# YOUR TASK:` comment with real values; do **not** copy a full solution from elsewhere):

Требуемая **структура** HCL (заполни каждый `___` и каждый комментарий `# YOUR TASK:` реальными значениями; **не** копируй готовое решение откуда-либо):

```hcl
# main.tf
terraform {
  required_version = ">= ___"                          # YOUR TASK: pin TF/OpenTofu floor
  required_providers {
    ___ = {                                             # YOUR TASK: provider local name
      source  = "___/___"                               # YOUR TASK: registry source
      version = "~> ___"                                # YOUR TASK: pin a major
    }
  }
}

provider "___" {
  region = var.region    # AWS shape; GCP needs `project` + `region` or `zone`,
                         # Yandex needs `zone`, kreuzwerker/docker takes no region.
                         # Adapt to your chosen provider — see Resources block.
  # ⛔ no inline credentials (access_key/secret_key, JSON SA key body, etc.) — env vars or profile only
}

# YOUR TASK: a data source that returns the latest Ubuntu 24.04 image ID
# (Hint per cloud: aws_ami / google_compute_image / yandex_compute_image).
# Hardcoded AMI IDs go stale within months — that's why this is a data source.
data "___" "ubuntu" {
  # filters / owners / family — YOUR TASK
}

# YOUR TASK: a security group / firewall named "lab4-sg" that allows:
#   - inbound 22/tcp from var.my_ip/32 only        (NEVER 0.0.0.0/0 for SSH)
#   - inbound 80/tcp and 5000/tcp from anywhere   (your app port)
#   - egress all
# Tag it { Name = "lab4-sg", Env = "lab4" }.
resource "___" "lab" {
  # name, description, ingress blocks, egress block, tags — YOUR TASK
}

# YOUR TASK: an SSH key resource — public key from var.ssh_public_key.
resource "___" "lab" {
  # name + public_key — YOUR TASK
}

# YOUR TASK: the VM itself. Required wiring:
#   - ami / image  ← from your data source above
#   - instance_type / machine_type ← var.instance_type
#   - key_name / metadata.ssh-keys ← your key resource
#   - vpc_security_group_ids / network_interface ← your SG/firewall
#   - user_data: a one-liner that runs on first boot and prints
#       "hello from $(hostname)" into /var/log/lab4-boot.log
#   - tags: { Name = "lab4-web", Env = "lab4" }
resource "___" "web" {
  # YOUR TASK
}
```

Пояснение: каркас задаёт порядок блоков — `terraform {}` с ограничениями версий, `provider`, `data` с последним образом Ubuntu, security group, SSH-ключ и саму VM. Все `___` и комментарии `# YOUR TASK:` нужно заменить рабочими значениями под выбранного провайдера.

```hcl
# variables.tf
# YOUR TASK: declare 4 variables — region, instance_type, my_ip, ssh_public_key.
# Use real HCL types (string / number / bool / list). Reminder: `int` is NOT a
# valid HCL type — use `number`. Mark ssh_public_key sensitive = true.
```

Пояснение: здесь объявляются четыре переменные — `region`, `instance_type`, `my_ip`, `ssh_public_key`; для чисел используется тип `number` (не `int`), а публичный ключ помечается `sensitive = true`.

```hcl
# outputs.tf
output "public_ip" {
  value = ___                                           # YOUR TASK: resource ref
}

output "ssh_command" {
  value = ___                                           # YOUR TASK: copy-paste-ready string
                                                        # e.g. "ssh -i ~/.ssh/lab04 ubuntu@<ip>"
}
```

Пояснение: `public_ip` берётся из атрибута созданного ресурса, а `ssh_command` собирается в готовую к копированию строку вида `ssh -i ~/.ssh/lab04 ubuntu@<ip>`.

```gitignore
# terraform/.gitignore
# YOUR TASK: ignore everything that holds state, secrets, or local downloads.
# Hint: tfstate, tfstate.backup, the .terraform/ dir, *.tfvars, *.pem, *.key.
# DO commit .terraform.lock.hcl — it pins provider versions for reproducibility.
```

Пояснение: игнорироваться должны все файлы state и его резервные копии, каталог `.terraform/`, файлы `*.tfvars`, приватные ключи `*.pem` и `*.key`; при этом `.terraform.lock.hcl` коммитится, потому что фиксирует версии провайдеров.

### 1.2 — Run the lifecycle (and read the plan)

1.2 — Прогоняем жизненный цикл (и читаем план)

```bash
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan      # READ THIS. + create / ~ update / - destroy / -/+ replace
terraform apply tfplan
```

Пояснение: `fmt` форматирует файлы, `init` скачивает провайдеры, `validate` проверяет синтаксис и ссылки, `plan` строит diff (`+` создать, `~` изменить, `-` удалить, `-/+` пересоздать) и сохраняет его в файл `tfplan`, а `apply` применяет именно сохранённый план.

`YOUR TASK`: after `apply`, **SSH in and prove the user_data ran**:

`YOUR TASK`: после `apply` **зайди по SSH и докажи, что user_data отработал**:

```bash
ssh -i ~/.ssh/lab04 ubuntu@<public_ip> "cat /var/log/lab4-boot.log; hostname"
```

(If you picked the docker-provider stand-in: `docker exec` instead and `curl` the app port — the reference submission shows the equivalent capture.)

(Если выбрал замену на docker-provider: используй `docker exec` и `curl` на порт приложения — в эталонной сдаче показано, как выглядит эквивалентный скриншот.)

### 1.3 — State management — the failure-mode focus

1.3 — Управление state — фокус на режимах отказа

This is where students and pros both lose hours. Lecture 4 slides 15–17 are the theory; this is the practice.

Именно здесь и студенты, и профессионалы теряют часы. Слайды 15–17 лекции 4 — теория; здесь — практика.

`YOUR TASK`: in `docs/LAB04.md`, paste the output of `terraform state list` and answer in **3 sentences total**:

`YOUR TASK`: в `docs/LAB04.md` вставь вывод `terraform state list` и ответь всего на **3 предложения**:

1. What does `terraform.tfstate` contain that makes it a secret? (Look at it. Don't commit it.)
   * Что лежит в `terraform.tfstate`, из-за чего этот файл считается секретом? (Загляни в него. Но не коммить.)
2. Why is local state catastrophic for a team of two engineers?
   * Почему локальный state катастрофичен для команды из двух инженеров?
3. What does `.terraform.lock.hcl` do, and why do you commit *it* but never the tfstate?
   * Что делает `.terraform.lock.hcl` и почему коммитят *его*, но никогда — tfstate?

### 1.4 — Proof of work

1.4 — Подтверждение работы

**Paste into `docs/LAB04.md`:**

**Вставь в `docs/LAB04.md`:**

- `terraform plan` output (sanitized — strip ARNs/account IDs)
  * вывод `terraform plan` (обезличенный — убери ARN и ID аккаунта)
- `terraform apply` summary line (`N to add, M to change, K to destroy`)
  * итоговая строка `terraform apply` (`N to add, M to change, K to destroy`)
- `terraform state list` output
  * вывод `terraform state list`
- The SSH session showing `cat /var/log/lab4-boot.log` and `hostname`
  * SSH-сессия с выводом `cat /var/log/lab4-boot.log` и `hostname`
- Output of `find terraform -maxdepth 2 -type f | sort` proving `.tfstate` is **not** in the tree
  * вывод `find terraform -maxdepth 2 -type f | sort`, доказывающий, что `.tfstate` **нет** в дереве файлов

---

## Task 2 — Pulumi VM (4 pts)

Задание 2 — VM на Pulumi (4 балла)

**Goal:** recreate the *same* infrastructure in Pulumi Python so you can compare it.

**Цель:** воссоздать *ту же* инфраструктуру на Pulumi Python, чтобы ты мог их сравнить.

### 2.1 — Tear down or keep the Terraform VM

2.1 — Снести или оставить Terraform VM

`YOUR TASK`: pick one path and document it:

`YOUR TASK`: выбери один путь и задокументируй его:

- **Path A** (recommended): `terraform destroy` first, build the Pulumi VM, keep that one for Lab 5. Paste the `destroy` output.
  * **Путь A** (рекомендуется): сначала `terraform destroy`, затем подними Pulumi VM и оставь её для лабы 5. Вставь вывод `destroy`.
- **Path B**: keep the Terraform VM for Lab 5, build the Pulumi VM, `pulumi destroy` it after the screenshot. Don't leave two VMs running undocumented — that's how billing alerts fire at 3 a.m.
  * **Путь B**: оставь Terraform VM для лабы 5, подними Pulumi VM, а после скриншота выполни `pulumi destroy`. Не оставляй две VM работать без документации — именно так в 3 часа ночи срабатывают алерты о списаниях.

### 2.2 — Write the Pulumi program

2.2 — Пишем программу на Pulumi

```python
# pulumi/__main__.py
# YOUR TASK: imports — pulumi + your provider package (pulumi_aws / pulumi_gcp / …)
import ___
import ___ as ___

# YOUR TASK: read the same three inputs you used in TF, via pulumi.Config().
# pulumi config set myIp $(curl -s ifconfig.me)
# pulumi config set sshPublicKey "$(cat ~/.ssh/lab04.pub)"
# pulumi config set instanceType t3.micro
cfg = ___
my_ip       = ___
ssh_pub     = ___
instance_ty = ___

# YOUR TASK: the same Ubuntu lookup as your TF data source,
# but using the provider's get_<image> function (e.g. aws.ec2.get_ami(...)).
ubuntu = ___

# YOUR TASK: the security group / firewall — same rules as TF.
sg = ___

# YOUR TASK: the key pair — same public key.
key = ___

# YOUR TASK: the instance itself. Pass the same user_data string.
web = ___

# YOUR TASK: two exports — public_ip and a ready-to-paste ssh command.
pulumi.export("public_ip", ___)
pulumi.export("ssh_command", ___)         # hint: .apply(lambda ip: f"ssh ...")
```

Пояснение: программа повторяет HCL — читает параметры через `pulumi.Config()`, находит образ Ubuntu функцией вида `get_ami`, создаёт security group, ключ и инстанс с тем же `user_data`, затем экспортирует публичный IP и готовую SSH-команду.

```ini
# pulumi/Pulumi.yaml
# YOUR TASK: project name lab04, runtime python, description one line.
```

Пояснение: в `Pulumi.yaml` указываются имя проекта `lab04`, рантайм `python` и описание в одну строку.

```text
# pulumi/requirements.txt
# YOUR TASK: pin pulumi >= 3.243 and your provider package
# (pulumi-aws / pulumi-gcp / pulumi-yandex / pulumi-docker).
```

Пояснение: зафиксируй `pulumi` не ниже 3.243 и пакет провайдера (`pulumi-aws`, `pulumi-gcp`, `pulumi-yandex` или `pulumi-docker`), чтобы окружение было воспроизводимым.

```gitignore
# pulumi/.gitignore
# YOUR TASK: ignore venv/, __pycache__/, and any Pulumi.<stack>.yaml that
# contains plaintext secrets. The stack name itself (Pulumi.dev.yaml) may be
# committed if all values are `pulumi config set --secret`-encrypted.
```

Пояснение: игнорируются виртуальное окружение и кэш Python, а также файлы стека с незашифрованными секретами; `Pulumi.dev.yaml` можно коммитить только при условии, что все значения зашифрованы через `pulumi config set --secret`.

### 2.3 — Run the lifecycle

2.3 — Прогоняем жизненный цикл

```bash
pulumi login --local                             # or `pulumi login` for Pulumi Cloud
pulumi stack init dev
pulumi config set aws:region eu-central-1        # adapt to your provider
pulumi config set myIp $(curl -s ifconfig.me)
pulumi config set --secret sshPublicKey "$(cat ~/.ssh/lab04.pub)"

pulumi preview                                   # ≈ terraform plan
pulumi up --yes
# ... ssh in and verify ...
pulumi destroy --yes                             # if Path B above, or before Lab 5 if Path A
```

Пояснение: `pulumi login --local` хранит state локально, `stack init dev` создаёт стек, `config set` задаёт параметры (секреты — с флагом `--secret`), `preview` — аналог `terraform plan`, `up` применяет изменения, а `destroy` всё удаляет.

### 2.4 — Proof of work

2.4 — Подтверждение работы

**Paste into `docs/LAB04.md`:**

**Вставь в `docs/LAB04.md`:**

- `pulumi preview` output (sanitized)
  * вывод `pulumi preview` (обезличенный)
- `pulumi up` summary line
  * итоговая строка `pulumi up`
- The SSH session against the Pulumi VM (or the equivalent on the docker stand-in)
  * SSH-сессия на Pulumi VM (или эквивалент на docker-замене)
- Either the `terraform destroy` proof (Path A) or the `pulumi destroy` proof (Path B)
  * подтверждение `terraform destroy` (путь A) либо `pulumi destroy` (путь B)

### 2.5 — The comparison (this is the point of doing both)

2.5 — Сравнение (ради этого и делаем оба варианта)

`YOUR TASK`: in `docs/LAB04.md`, write 3–5 sentences for each of:

`YOUR TASK`: в `docs/LAB04.md` напиши по 3–5 предложений для каждого пункта:

1. **Readability** — which file was easier to skim a week later?
   * **Читаемость** — какой файл было легче просмотреть через неделю?
2. **Logic & loops** — imagine deploying the same VM × `["dev","staging","prod"]`. Which tool did you find more natural for that?
   * **Логика и циклы** — представь, что ту же VM нужно развернуть × `["dev","staging","prod"]`. Какой инструмент показался тебе естественнее для этого?
3. **Debugging** — when your `apply`/`up` failed, which tool's error message pointed you at the fix faster?
   * **Отладка** — когда `apply`/`up` падал, сообщение об ошибке какого инструмента быстрее выводило тебя на исправление?
4. **State + secrets** — where does each tool put state? Which one encrypts secrets by default?
   * **State и секреты** — куда каждый инструмент кладёт state? Какой из них шифрует секреты по умолчанию?
5. **When you'd pick each** — one concrete scenario per tool, drawn from your own experience this week.
   * **Когда что выбирать** — по одному конкретному сценарию для каждого инструмента из твоего личного опыта за эту неделю.

---

## Task 3 — Documentation (2 pts)

Задание 3 — Документация (2 балла)

`docs/LAB04.md` must contain, in order:

`docs/LAB04.md` должен содержать по порядку:

1. **Provider & target** — which cloud (or docker stand-in), which region/zone, instance size, and a one-line cost note (should be `$0`).
   * **Провайдер и цель** — какое облако (или docker-замена), какой регион/зона, размер инстанса и одна строка о стоимости (должно быть `$0`).
2. **Terraform implementation** — version used, file layout, the 3-sentence state-management answer from 1.3, sanitized lifecycle output, SSH proof.
   * **Реализация на Terraform** — использованная версия, структура файлов, ответ про state из 1.3 в три предложения, обезличенный вывод жизненного цикла, подтверждение по SSH.
3. **Pulumi implementation** — language + version, lifecycle output, SSH proof.
   * **Реализация на Pulumi** — язык и версия, вывод жизненного цикла, подтверждение по SSH.
4. **Terraform vs Pulumi** — the 5-bullet comparison from 2.5.
   * **Terraform против Pulumi** — сравнение из пяти пунктов из 2.5.
5. **Lab 5 prep & cleanup** — which VM (if any) is staying alive, or how Lab 5 will recreate one from your IaC. Plus `destroy` proof for everything you tore down.
   * **Подготовка к лабе 5 и уборка** — какая VM (если вообще какая-то) остаётся работать или как лаба 5 воссоздаст её из твоего IaC. Плюс подтверждение `destroy` для всего, что ты снёс.

> All terminal output must be sanitized: no full account IDs, no access keys, no API tokens. Mark anything reconstructed `(illustrative)`.
>
> Весь вывод терминала должен быть обезличен: без полных ID аккаунтов, ключей доступа и API-токенов. Реконструированные фрагменты помечай `(illustrative)`.

---

## Bonus Task — IaC CI/CD + Remote State (2 pts)

Бонусное задание — CI/CD для IaC + удалённый state (2 балла)

### Part 1 — GitHub Actions validation (1 pt)

Часть 1 — Валидация в GitHub Actions (1 балл)

`YOUR TASK`: create `.github/workflows/terraform-ci.yml` that runs on PRs touching `terraform/**` and runs `fmt -check -recursive`, `init -backend=false`, `validate`, and `tflint`. Use **first-party** actions only — `hashicorp/setup-terraform` (or `opentofu/setup-opentofu`) and `terraform-linters/setup-tflint`. No long-lived cloud creds: `-backend=false` skips state.

`YOUR TASK`: создай `.github/workflows/terraform-ci.yml`, который запускается на PR, затрагивающих `terraform/**`, и выполняет `fmt -check -recursive`, `init -backend=false`, `validate` и `tflint`. Используй только **официальные** actions — `hashicorp/setup-terraform` (или `opentofu/setup-opentofu`) и `terraform-linters/setup-tflint`. Никаких долгоживущих облачных ключей: `-backend=false` пропускает state.

Skeleton — fill the blanks:

Каркас — заполни пропуски:

```yaml
# .github/workflows/terraform-ci.yml
name: terraform-ci
on:
  pull_request:
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform-ci.yml'

jobs:
  validate:
    runs-on: ubuntu-24.04
    defaults:
      run: { working-directory: terraform }
    steps:
      - uses: actions/checkout@v4
      - uses: ___                                       # YOUR TASK: setup-terraform action + version pin
        with:
          terraform_version: "___"                      # YOUR TASK: the version from the table above
      # YOUR TASK: 4 steps — fmt -check -recursive, init -backend=false, validate, tflint.
      # tflint needs its own setup action; add a minimal terraform/.tflint.hcl too.
```

Пояснение: workflow срабатывает только при изменениях в `terraform/**` и в самом файле workflow; шаг `setup-terraform` закрепляет нужную версию Terraform, дальше идут четыре проверки — форматирование, инициализация без backend, `validate` и линтер `tflint`, для которого нужен отдельный setup-action и минимальный `terraform/.tflint.hcl`.

**Prove it:**

**Докажи это:**

- Open a PR touching `terraform/` — the workflow runs and stays green.
  * Открой PR с изменением в `terraform/` — workflow запускается и остаётся зелёным.
- Push a `fmt`-violating commit on top — the workflow goes **red** at the `fmt -check` step. Screenshot or paste the failing log.
  * Добавь сверху коммит с нарушением `fmt` — workflow становится **красным** на шаге `fmt -check`. Приложи скриншот или вставь лог падения.
- Touch a non-`terraform/**` file — the workflow does **not** trigger.
  * Измени файл вне `terraform/**` — workflow **не** должен запуститься.

### Part 2 — Migrate to remote, locked state (1 pt)

Часть 2 — Переход на удалённый state с блокировкой (1 балл)

`YOUR TASK`:

`YOUR TASK` — твоё задание:

1. Create a state bucket: S3 with **versioning + encryption**, or GCS with versioning, or Yandex Object Storage. By hand or by a tiny separate Terraform config — document which.
   * Создай bucket для state: S3 с **версионированием и шифрованием**, либо GCS с версионированием, либо Yandex Object Storage. Вручную или отдельной крошечной конфигурацией Terraform — задокументируй, как именно.
2. Add a `backend "s3"` / `backend "gcs"` block to your `terraform/` config:
   * Добавь блок `backend "s3"` / `backend "gcs"` в конфигурацию `terraform/`:
   - For S3, use `use_lockfile = true` (TF 1.10+ native state locking — no more DynamoDB table).
     * Для S3 используй `use_lockfile = true` (нативная блокировка state в TF 1.10+ — таблица DynamoDB больше не нужна).
   - For GCS, locking is native and automatic.
     * В GCS блокировка нативная и автоматическая.
   - **Always** set `encrypt = true`.
     * **Всегда** указывай `encrypt = true`.
3. `terraform init -migrate-state` and confirm the local `terraform.tfstate` moved to the bucket.
   * Выполни `terraform init -migrate-state` и убедись, что локальный `terraform.tfstate` переехал в bucket.
4. Run `terraform plan` again — it must report **No changes** (proves the state migrated cleanly, not re-imported).
   * Снова запусти `terraform plan` — он должен сообщить **No changes** (это доказывает, что state перенёсся чисто, а не импортировался заново).
5. In `docs/LAB04.md`, explain in 2–3 sentences **why locking prevents two engineers from corrupting state** and what the failure mode looks like without it.
   * В `docs/LAB04.md` объясни в 2–3 предложениях, **почему блокировка не даёт двум инженерам испортить state** и как выглядит сбой без неё.

---

## How to Submit

Как сдавать

```bash
git switch -c lab04
git add terraform/ pulumi/ docs/LAB04.md
git add .github/workflows/terraform-ci.yml      # bonus only
git commit -m "feat(lab04): IaC with Terraform + Pulumi"
git push -u origin lab04
```

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab04` → `course-repo:master` *(reviewed)*
  * `your-fork:lab04` → `course-repo:master` *(с ревью)*
- `your-fork:lab04` → `your-fork:master`
  * `your-fork:lab04` → `your-fork:master`

PR checklist:

Чек-лист PR:

```text
- [ ] terraform/ provisions a VM end-to-end; SSH proven
- [ ] pulumi/ recreates the same infra; SSH proven
- [ ] No *.tfstate, *.tfvars, .terraform/, or Pulumi secret stack file in the diff
- [ ] .terraform.lock.hcl IS committed
- [ ] docs/LAB04.md has all 5 sections + the TF-vs-Pulumi comparison
- [ ] At most ONE VM left running, and it's named in docs (for Lab 5)
- [ ] Bonus (optional): terraform-ci.yml proven green+red; remote state migrated
```

Пояснение к чек-листу: Terraform создаёт VM от начала до конца, и SSH подтверждён; Pulumi воссоздаёт ту же инфраструктуру, и SSH тоже подтверждён; в diff нет `*.tfstate`, `*.tfvars`, каталога `.terraform/` и файла стека Pulumi с секретами; при этом `.terraform.lock.hcl` закоммичен; в `docs/LAB04.md` есть все 5 разделов и сравнение Terraform с Pulumi; работает не более одной VM, и она названа в документации (для лабы 5); бонус (по желанию): `terraform-ci.yml` показан и зелёным, и красным, а state перенесён в удалённый backend.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Terraform (4 pts)

Задание 1 — Terraform (4 балла)

- ✅ Provider chosen, authenticated via env/profile (no inline creds)
  * ✅ Провайдер выбран, аутентификация через env/профиль (без встроенных креденшелов)
- ✅ `data` source for the Ubuntu image (not a hardcoded AMI/ID)
  * ✅ Источник `data` для образа Ubuntu (не захардкоженный AMI/ID)
- ✅ Security group / firewall restricts **SSH 22 to your IP/32** only
  * ✅ Security group / firewall разрешает **SSH 22 только с твоего IP/32**
- ✅ All six primitives present: `terraform{}`, `provider`, `variable`, `resource`, `data`, `output`
  * ✅ Присутствуют все шесть примитивов: `terraform{}`, `provider`, `variable`, `resource`, `data`, `output`
- ✅ `user_data` runs on first boot and writes to `/var/log/lab4-boot.log`
  * ✅ `user_data` выполняется при первой загрузке и пишет в `/var/log/lab4-boot.log`
- ✅ SSH session in `docs/LAB04.md` shows the boot log line + hostname
  * ✅ SSH-сессия в `docs/LAB04.md` показывает строку из лога загрузки и hostname
- ✅ `.gitignore` correct: tfstate ignored, `.terraform.lock.hcl` committed
  * ✅ `.gitignore` настроен верно: tfstate игнорируется, `.terraform.lock.hcl` закоммичен
- ✅ 3-sentence state-management answer present
  * ✅ Ответ про управление state из трёх предложений присутствует

### Task 2 — Pulumi (4 pts)

Задание 2 — Pulumi (4 балла)

- ✅ Same infra recreated in Pulumi Python (or TS/Go/Java — Python recommended)
  * ✅ Та же инфраструктура воссоздана на Pulumi Python (или TS/Go/Java — рекомендуется Python)
- ✅ `pulumi preview` + `pulumi up` succeed; SSH proven
  * ✅ `pulumi preview` и `pulumi up` проходят успешно; SSH подтверждён
- ✅ At most one of the two VMs is left running and it's documented
  * ✅ Работает не более одной из двух VM, и это задокументировано
- ✅ 5-bullet TF-vs-Pulumi comparison
  * ✅ Сравнение Terraform и Pulumi из пяти пунктов

### Task 3 — Docs (2 pts)

Задание 3 — Документация (2 балла)

- ✅ All 5 sections in `docs/LAB04.md`
  * ✅ Все 5 разделов в `docs/LAB04.md`
- ✅ Outputs sanitized; cleanup proof included
  * ✅ Выводы обезличены; подтверждение уборки приложено

### Bonus (2 pts)

Бонус (2 балла)

- ✅ `terraform-ci.yml` runs `fmt`/`init`/`validate`/`tflint`, gated to `terraform/**`, proven green + red — **1 pt**
  * ✅ `terraform-ci.yml` запускает `fmt`/`init`/`validate`/`tflint`, ограничен `terraform/**`, доказан зелёным и красным — **1 балл**
- ✅ Remote, encrypted, **locked** backend; migration confirmed; locking rationale explained — **1 pt**
  * ✅ Удалённый, зашифрованный backend **с блокировкой**; перенос подтверждён; объяснение блокировки приведено — **1 балл**

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Terraform VM | **4** | Working infra, data source, restricted SSH, vars/outputs, clean state hygiene |
| **Task 2** — Pulumi VM | **4** | Same infra in Pulumi, SSH proven, comparison written |
| **Task 3** — Documentation | **2** | All 5 sections, sanitized output, cleanup proof |
| **Bonus** — CI for IaC | **1** | Path-filtered fmt/validate/lint workflow, green-and-red proof |
| **Bonus** — Remote state | **1** | Encrypted + locked backend, migration proven, rationale explained |
| **Total** | **10 + 2** | 10 main + 2 bonus |

| Задание | Баллы | Критерий |
|------|-------:|----------|
| **Задание 1** — VM на Terraform | **4** | Работающая инфраструктура, data source, ограниченный SSH, переменные и outputs, чистое управление state |
| **Задание 2** — VM на Pulumi | **4** | Та же инфраструктура на Pulumi, подтверждённый SSH, написанное сравнение |
| **Задание 3** — Документация | **2** | Все 5 разделов, обезличенный вывод, подтверждение уборки |
| **Бонус** — CI для IaC | **1** | Workflow с фильтром по путям для fmt/validate/lint, доказательства зелёного и красного |
| **Бонус** — Удалённый state | **1** | Зашифрованный backend с блокировкой, подтверждённый перенос, объяснение причин |
| **Итого** | **10 + 2** | 10 основных + 2 бонусных |

**Non-negotiables:** free-tier only · no secrets or state in Git · SSH proof required · at most one VM left for Lab 5.

**Жёсткие требования:** только бесплатный тариф · никаких секретов и state в Git · обязательное подтверждение по SSH · не более одной VM, оставленной для лабы 5.

---

## Resources

Ресурсы

<details>
<summary>📚 Terraform / OpenTofu</summary>

📚 Terraform / OpenTofu — материалы и документация

- [Terraform docs](https://developer.hashicorp.com/terraform/docs) · [Registry](https://registry.terraform.io/)
  * [Документация Terraform](https://developer.hashicorp.com/terraform/docs) · [Registry](https://registry.terraform.io/)
- [OpenTofu](https://opentofu.org) — drop-in fork; migration guide
  * [OpenTofu](https://opentofu.org) — совместимый форк (drop-in); руководство по миграции
- [S3 backend](https://developer.hashicorp.com/terraform/language/settings/backends/s3) · [State locking](https://developer.hashicorp.com/terraform/language/state/locking) · [`terraform import`](https://developer.hashicorp.com/terraform/cli/import)
  * [S3 backend](https://developer.hashicorp.com/terraform/language/settings/backends/s3) · [Блокировка state](https://developer.hashicorp.com/terraform/language/state/locking) · [`terraform import`](https://developer.hashicorp.com/terraform/cli/import)
- [tflint](https://github.com/terraform-linters/tflint)
  * [tflint](https://github.com/terraform-linters/tflint) — линтер для Terraform
- *Terraform: Up & Running* — Brikman (4e, 2024 — covers OpenTofu)
  * *Terraform: Up & Running* — Брикман (4-е изд., 2024 — рассматривает и OpenTofu)

</details>

<details>
<summary>📦 Pulumi</summary>

📦 Pulumi — материалы и документация

- [Pulumi docs](https://www.pulumi.com/docs/) · [Registry](https://www.pulumi.com/registry/)
  * [Документация Pulumi](https://www.pulumi.com/docs/) · [Registry](https://www.pulumi.com/registry/)
- [Pulumi vs Terraform](https://www.pulumi.com/docs/concepts/vs/terraform/) · [Secrets](https://www.pulumi.com/docs/concepts/secrets/)
  * [Pulumi против Terraform](https://www.pulumi.com/docs/concepts/vs/terraform/) · [Секреты](https://www.pulumi.com/docs/concepts/secrets/)

</details>

<details>
<summary>☁️ Provider quick-references</summary>

☁️ Краткие справки по провайдерам

- **AWS** — [provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs); auth via `AWS_ACCESS_KEY_ID` / `~/.aws/credentials`. Ubuntu owner ID: `099720109477`.
  * **AWS** — [документация провайдера](https://registry.terraform.io/providers/hashicorp/aws/latest/docs); аутентификация через `AWS_ACCESS_KEY_ID` / `~/.aws/credentials`. ID владельца Ubuntu: `099720109477`.
- **GCP** — [provider docs](https://registry.terraform.io/providers/hashicorp/google/latest/docs); auth via `gcloud auth application-default login`. Enable Compute Engine API. SSH key in instance `metadata = { ssh-keys = "ubuntu:${var.ssh_public_key}" }`.
  * **GCP** — [документация провайдера](https://registry.terraform.io/providers/hashicorp/google/latest/docs); аутентификация через `gcloud auth application-default login`. Включи Compute Engine API. SSH-ключ задаётся в `metadata = { ssh-keys = "ubuntu:${var.ssh_public_key}" }` инстанса.
- **Yandex** — [provider docs](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs); service-account JSON key. Free-tier shape: `standard-v3`, `core_fraction = 20`, 1 GB RAM.
  * **Yandex** — [документация провайдера](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs); JSON-ключ сервисного аккаунта. Конфигурация бесплатного тарифа: `standard-v3`, `core_fraction = 20`, 1 ГБ RAM.
- **Docker stand-in** — `kreuzwerker/docker ~> 3.0`; `docker_image` + `docker_container`. No SSH; `docker exec` + curl the published port.
  * **Замена на Docker** — `kreuzwerker/docker ~> 3.0`; `docker_image` + `docker_container`. Без SSH; `docker exec` и `curl` на опубликованный порт.

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Типичные ошибки (по реальным прогонам)

- **`type = int` is invalid HCL.** Variable types are `string`, `number`, `bool`, `list`, `map`, `set`, `object`, `tuple`, `any`. `int` will fail `terraform validate` with a "Invalid type specification" — use `number`.
  * **`type = int` — недопустимый HCL.** Типы переменных: `string`, `number`, `bool`, `list`, `map`, `set`, `object`, `tuple`, `any`. `int` уронит `terraform validate` с ошибкой «Invalid type specification» — используй `number`.
- **Committed `terraform.tfstate`.** Holds resource IDs, private IPs, sometimes plaintext secrets, and merge conflicts on JSON are *unrecoverable*. `.gitignore` line 1.
  * **Закоммиченный `terraform.tfstate`.** В нём ID ресурсов, приватные IP, иногда секреты в открытом виде, а конфликты слияния в JSON *невосстановимы*. Первая строка `.gitignore`.
- **Forgetting `.terraform.lock.hcl`.** This one **you DO commit** — it pins provider hashes so a teammate's `init` resolves to the same `aws ~> 5.x` patch you used. Without it, two engineers can apply with subtly different provider versions and produce different infra.
  * **Забыть про `.terraform.lock.hcl`.** Этот файл **нужно коммитить** — он фиксирует хэши провайдеров, чтобы `init` у коллеги разрешился в тот же патч `aws ~> 5.x`, что и у тебя. Без него два инженера могут применить конфигурацию с чуть разными версиями провайдера и получить разную инфраструктуру.
- **Credentials in `provider {}`.** Code Spaces (2014) was deleted in 12 hours because of this exact pattern. Env vars or shared profile, never inline. Don't put them in `.tfvars` either — that's another file students commit by accident.
  * **Учётные данные в `provider {}`.** Code Spaces (2014) прекратил существование за 12 часов именно из-за такого шаблона. Только переменные окружения или общий профиль, никогда не вписывай внутрь. Не клади их и в `.tfvars` — студенты часто коммитят этот файл случайно.
- **Default-deny egress on a new AWS security group.** Terraform's `aws_security_group` resource has *no implicit egress* unlike the AWS console — you must write the `egress` block yourself or the VM can't reach the internet for `apt`, the user_data hangs, and `apply` "succeeds" with a broken host.
  * **Запрет исходящего трафика по умолчанию у новой AWS security group.** У ресурса `aws_security_group` в Terraform *нет неявного egress*, в отличие от консоли AWS — блок `egress` нужно написать самому, иначе VM не сможет выйти в интернет для `apt`, user_data зависнет, а `apply` «успешно» завершится, оставив сломанный хост.
- **Relying on the default VPC.** It exists on day-one AWS accounts but new orgs delete it for compliance. Either look it up via `data "aws_vpc" "default" { default = true }` or create your own — never assume.
  * **Надежда на VPC по умолчанию.** Он есть в новых AWS-аккаунтах, но новые организации удаляют его ради соответствия требованиям. Либо найди его через `data "aws_vpc" "default" { default = true }`, либо создай свой — не рассчитывай на него.
- **SSH wide open.** `0.0.0.0/0` on port 22 = your VM is in a botnet within hours. `var.my_ip/32` only. (`curl -s ifconfig.me` to find your IP.)
  * **SSH открыт всему миру.** `0.0.0.0/0` на порту 22 = твоя VM окажется в ботнете через считаные часы. Только `var.my_ip/32`. (`curl -s ifconfig.me`, чтобы узнать свой IP.)
- **Hardcoded AMI ID.** `ami-0c55b159cbfafe1f0` was the Ubuntu 22.04 AMI in eu-central-1 *once*. AMIs rotate as Canonical ships patches; hardcoded IDs go 404 in months. Use a `data "aws_ami"` filter on the name pattern.
  * **Захардкоженный AMI ID.** `ami-0c55b159cbfafe1f0` *когда-то* был AMI Ubuntu 22.04 в eu-central-1. AMI меняются по мере выхода патчей Canonical; захардкоженные ID отваливаются с 404 через несколько месяцев. Используй фильтр `data "aws_ami"` по шаблону имени.
- **Two simultaneous `apply`s with local state.** State JSON gets half-written, Terraform can't parse it, recovery is hand-editing — which itself is forbidden. This is the whole reason the bonus task exists.
  * **Два одновременных `apply` с локальным state.** JSON state записывается частично, Terraform не может его разобрать, а восстановление — ручная правка, которая сама по себе запрещена. Ради этого и существует бонусное задание.

</details>

---

## Looking Ahead

Что дальше

| Lab | What it does with your Lab 4 VM |
|---:|---|
| 5 | Ansible roles SSH in and install Docker + your Lab 1–3 app |
| 6 | Ansible blocks/rescue, tags, Compose deploy via the same SSH path |
| 9 | k3d replaces the hand-VM model — same IaC mindset, different unit |
| 13 | ArgoCD takes over deploys; your IaC still defines the cluster |

| Лаба | Что она делает с твоей VM из лабы 4 |
|---:|---|
| 5 | Ansible-роли заходят по SSH и ставят Docker и твоё приложение из лаб 1–3 |
| 6 | Блоки `block`/`rescue`, теги, деплой через Compose по тому же SSH-пути |
| 9 | k3d заменяет модель ручной VM — тот же подход IaC, другая единица |
| 13 | ArgoCD берёт деплои на себя; твой IaC по-прежнему описывает кластер |

> **Remember:** if you can't `git clone && terraform apply` from scratch, it's not IaC — it's a history of console clicks. No secrets in code, no state in Git. The five rules from lecture 4 slide 22 are paid for in real money (slide 23).
>
> **Запомни:** если ты не можешь с нуля выполнить `git clone && terraform apply`, то это не IaC, а история кликов в консоли. Никаких секретов в коде, никакого state в Git. Пять правил со слайда 22 лекции 4 оплачены реальными деньгами (слайд 23).
