# 📌 Lecture 4 — Infrastructure as Code with Terraform & Pulumi

Лекция 4 — Инфраструктура как код с Terraform и Pulumi

## 📍 Slide 1 – 🏗️ Welcome to Infrastructure as Code

Слайд 1 — Добро пожаловать в инфраструктуру как код

* 🐶 **Lecture 1 promised cattle, not pets** — today we deliver
  * 🐶 **Лекция 1 обещала скот, а не питомцев** — сегодня выполняем обещание
* 📝 **IaC** = your cloud lives in Git, not in someone's browser tabs
  * 📝 **IaC** = твоё облако живёт в Git, а не во вкладках браузера у кого-то другого
* 🌍 **Terraform + Pulumi** = the two ways modern teams provision infrastructure
  * 🌍 **Terraform + Pulumi** — два подхода, которыми современные команды создают инфраструктуру
* 🎯 By the end: you can spin up (and tear down) a VM with one command — and review the diff before applying
  * 🎯 К концу лекции ты сможешь поднять (и снести) VM одной командой — и посмотреть diff перед apply

```mermaid
flowchart LR
  Manual[🖱️ Console clicks] -->|IaC| Code[📝 Code in Git]
  Code --> Plan[📋 plan]
  Plan --> Apply[🚀 apply]
  Apply --> Cloud[☁️ Reproducible cloud]
```

Схема: ручные клики в консоли заменяются кодом в Git; код проходит через plan и apply и превращается в воспроизводимое облако.

> 🔗 **Tie-in to Lab 4:** you'll build the *same* VM twice — once with Terraform, once with Pulumi — and feel the trade-offs first hand. The bonus task adds a GitHub Actions workflow that runs `plan` on every PR.
>
> 🔗 **Связь с лабой 4:** ты соберёшь *одну и ту же* VM дважды — сначала на Terraform, потом на Pulumi — и на практике почувствуешь разницу подходов. Бонусное задание добавляет workflow GitHub Actions, который запускает `plan` на каждом PR.

---

## 📍 Slide 2 – 🎯 Learning Outcomes

Слайд 2 — Результаты обучения

| # | Outcome |
|---|---------|
| 1 | 🧠 Explain why snowflake servers are a business risk |
| 2 | 📝 Write a working Terraform configuration with provider, resource, variable, output |
| 3 | 🔄 Run the `init → plan → apply → destroy` lifecycle confidently |
| 4 | 📦 Move state from local disk to a remote, locked backend — and explain why |
| 5 | 🐍 Provision the same infrastructure with Pulumi in Python |
| 6 | ⚖️ Pick Terraform vs Pulumi vs OpenTofu based on team and use case |

| # | Результат |
|---|-----------|
| 1 | 🧠 Объяснить, почему уникальные серверы — это бизнес-риск |
| 2 | 📝 Написать рабочую конфигурацию Terraform с provider, resource, variable и output |
| 3 | 🔄 Уверенно пройти жизненный цикл `init → plan → apply → destroy` |
| 4 | 📦 Перенести state с локального диска в удалённый backend с блокировками — и объяснить зачем |
| 5 | 🐍 Развернуть ту же инфраструктуру на Pulumi с Python |
| 6 | ⚖️ Выбирать между Terraform, Pulumi и OpenTofu с учётом команды и задачи |

---

## 📍 Slide 3 – 🔧 Tech Stack Pinned for May 2026

Слайд 3 — Технологический стек, зафиксированный на май 2026

| Tool | Version | Notes |
|------|---------|-------|
| 🌍 **Terraform** | **1.15.3** | Released May 13 2026; BSL-1.1 since **August 10, 2023** |
| 🌱 **OpenTofu** | **1.12.0** | Released May 14 2026; MPL-2.0, Linux Foundation; 1:1 drop-in CLI |
| 📦 **Pulumi** | **3.243.0** | May 2026; Apache-2.0 throughout |
| ☁️ **AWS provider** | v5.x | Used in most Lab 4 examples |
| ☁️ **GCP provider** | v6.x | Alternative for Lab 4 |
| ☁️ **Yandex provider** | v0.115+ | Default for Russia-only labs |

| Инструмент | Версия | Примечания |
|------------|--------|------------|
| 🌍 **Terraform** | **1.15.3** | Выпущен 13 мая 2026; BSL-1.1 с **10 августа 2023** |
| 🌱 **OpenTofu** | **1.12.0** | Выпущен 14 мая 2026; MPL-2.0, Linux Foundation; CLI-совместим 1:1 |
| 📦 **Pulumi** | **3.243.0** | Май 2026; Apache-2.0 по всему стеку |
| ☁️ **AWS provider** | v5.x | Используется в большинстве примеров лабы 4 |
| ☁️ **GCP provider** | v6.x | Альтернатива для лабы 4 |
| ☁️ **Yandex provider** | v0.115+ | По умолчанию для лаб, рассчитанных только на Россию |

> 🔧 **You can swap `terraform` for `tofu` in every command on these slides.** The HCL is identical. Choose based on license preference and team norms — labs accept both.
>
> 🔧 **В любой команде на этих слайдах ты можешь заменить `terraform` на `tofu`.** HCL при этом не меняется. Выбирай по лицензионным предпочтениям и нормам команды — лабы принимают оба варианта.

---

## 📍 Slide 4 – ❓ The Big Question

Слайд 4 — Большой вопрос

* 📊 The 2024 Flexera *State of the Cloud* report: **89%** of enterprises run multi-cloud — and **70%+** report uncontrolled drift between intended and actual config
  * 📊 Отчёт Flexera *State of the Cloud* за 2024 год: **89%** предприятий работают в multi-cloud — и **свыше 70%** сообщают о неконтролируемом дрейфе между задуманной и фактической конфигурацией
* ⏱️ Manual VM provisioning: **hours to days** for one server, then days more to make a copy match
  * ⏱️ Ручное развёртывание VM: **от часов до дней** на один сервер, а потом ещё дни, чтобы копия стала его точным подобием
* 💥 Most outages are caused by **changes** — and untracked changes are the worst kind
  * 💥 Большинство аварий вызвано **изменениями** — а неучтённые изменения самые опасные

> 💬 *"It worked in staging — staging was different."* — Every ops engineer, ever
>
> 💬 *«В стейджинге работало — просто стейджинг был другим»* — каждый инженер эксплуатации в истории.

**🤔 Think:**

**🤔 Подумай:**

* Could you recreate your production cloud from scratch tomorrow?
  * Смог бы ты завтра с нуля воссоздать своё продовое облако?
* If the person who built it left this morning, how long until you're stuck?
  * Если человек, который его построил, ушёл сегодня утром, сколько пройдёт до момента, когда ты застрянешь?

---

## 📍 Slide 5 – 🐶 Pets vs. Cattle (Bill Baker, Microsoft, 2012)

Слайд 5 — Питомцы и скот (Билл Бейкер, Microsoft, 2012)

* 🐶 **Pets** — named (`web-prod-01`), nursed, irreplaceable, downtime if sick
  * 🐶 **Питомцы** — у каждого есть имя (`web-prod-01`), их холят и лечат, они незаменимы; заболел — жди простоя
* 🐄 **Cattle** — numbered (`web-001..099`), identical, disposable, replaced if sick
  * 🐄 **Скот** — пронумерован (`web-001..099`), все одинаковые, расходный материал; заболел — заменили
* ☁️ **Cloud-native = cattle mindset.** Lecture 1 foreshadowed this; today we make it real.
  * ☁️ **Cloud-native = мышление скота.** Лекция 1 только предвещала это; сегодня делаем это реальностью.

```mermaid
flowchart LR
  Pets[🐶 Pets<br/>SSH'd into, hand-tuned]
  Cattle[🐄 Cattle<br/>Defined in code, replaced in minutes]
  Pets -.->|IaC + immutable infra| Cattle
```

Схема: питомцев, в которых заходят по SSH и которых правят руками, заменяет скот — определённый в коде и заменяемый за минуты благодаря IaC и неизменяемой инфраструктуре.

> 📖 Bill Baker coined this metaphor in a 2012 internal Microsoft talk later popularized by Randy Bias of CloudScaling. Twelve years later, every cloud architecture book still uses it. It works because it's true.
>
> 📖 Билл Бейкер придумал эту метафору во внутреннем докладе в Microsoft в 2012 году, а позже её популяризировал Рэнди Байас из CloudScaling. Двенадцать лет спустя её использует каждая книга об облачной архитектуре. Она работает, потому что правдива.

---

## 📍 Slide 6 – 😱 Snowflakes, Drift, and Bus Factor

Слайд 6 — Уникальные серверы, дрейф и bus factor

The three failure modes IaC fixes:

Три режима отказа, которые исправляет IaC:

| Failure | What it looks like | What it costs |
|---------|---------------------|---------------|
| ❄️ **Snowflake server** | "Don't touch web-03, only Petya knows what's on it" | Can't scale, can't reproduce, can't replace |
| 📉 **Configuration drift** | Staging and prod were identical 6 months ago — not anymore | Bugs that only repro in prod, hours debugging |
| 🚌 **Bus factor = 1** | One person holds the cloud architecture in their head | Project halts when they're on vacation or quit |

| Сбой | Как это выглядит | Чем обходится |
|------|------------------|---------------|
| ❄️ **Уникальный сервер (snowflake)** | «Не трогай web-03, только Петя знает, что там» | Не масштабируется, не воспроизводится, не заменяется |
| 📉 **Дрейф конфигурации** | Полгода назад стейджинг и прод были одинаковыми — уже нет | Баги, которые воспроизводятся только в проде, и часы отладки |
| 🚌 **Bus factor = 1** | Архитектура облака живёт в голове одного человека | Проект встаёт, когда он в отпуске или уволился |

> 🔥 **Hot take:** if you can't *delete and recreate* your production environment from a Git tag, you don't have IaC — you have an `aws` CLI history.
>
> 🔥 **Резко, но честно:** если ты не можешь *удалить и заново создать* продовое окружение из Git-тега, у тебя не IaC, а история команд `aws` CLI.

---

## 📍 Slide 7 – 💡 What Infrastructure as Code Actually Is

Слайд 7 — Что такое инфраструктура как код на самом деле

* 📝 **Definition:** managing and provisioning infrastructure through *machine-readable, version-controlled* config files instead of manual console clicks or ad-hoc scripts.
  * 📝 **Определение:** управление инфраструктурой и её создание через *читаемые машиной и хранящиеся под контролем версий* конфиг-файлы вместо ручных кликов в консоли или ad-hoc-скриптов.
* 🔄 The cloud becomes a **build artifact**, not a hand-crafted environment
  * 🔄 Облако становится **артефактом сборки**, а не средой, собранной руками
* 📜 Every change goes through the same process you use for application code: PR → review → CI → merge → deploy
  * 📜 Каждое изменение проходит тот же процесс, что и код приложения: PR → ревью → CI → merge → деплой

```mermaid
flowchart LR
  Code[📝 .tf / .py in Git] --> CI[🧪 CI: plan]
  CI --> Review[👀 PR review]
  Review --> Merge[🚀 Merge → apply]
  Merge --> Cloud[☁️ Real infrastructure]
  Cloud -->|diff| Code
```

Схема: файлы `.tf` или `.py` лежат в Git, CI строит plan, изменение проходит ревью и merge, после чего apply приводит облако в соответствие с кодом; расхождения возвращаются в код как diff.

> 📖 *Infrastructure as Code* — Kief Morris (2nd ed., 2020). The reference text; defines the four core principles below.
>
> 📖 *Infrastructure as Code* — Киф Моррис (2-е изд., 2020). Эталонная книга; в ней сформулированы четыре принципа, о которых пойдёт речь ниже.

---

## 📍 Slide 8 – 📐 The Four Principles (Morris)

Слайд 8 — Четыре принципа (Моррис)

1. ♻️ **Reproducibility** — same code → same infrastructure, every time
   * ♻️ **Воспроизводимость** — один и тот же код → одна и та же инфраструктура, каждый раз
2. 🔁 **Disposability** — destroy and rebuild should be routine, not heroic
   * 🔁 **Одноразовость** — уничтожить и пересобрать должно быть рутиной, а не подвигом
3. 📜 **Consistency** — dev, staging, prod differ only in variables
   * 📜 **Согласованность** — dev, staging и prod отличаются только переменными
4. 🔍 **Visibility** — `git log` answers "who changed what, when, why"
   * 🔍 **Прозрачность** — `git log` отвечает на вопрос «кто, что, когда и зачем изменил»

> 💡 **If any one of these is missing, you have automation — not IaC.** Bash scripts that create a VM are not IaC; they're a recipe nobody can replay safely.
>
> 💡 **Если нет хотя бы одного из этих принципов, у тебя автоматизация, а не IaC.** Bash-скрипты, создающие VM, — не IaC; это рецепт, который никто не сможет безопасно повторить.

---

## 📍 Slide 9 – 🔀 Declarative vs. Imperative

Слайд 9 — Декларативный и императивный подходы

```mermaid
flowchart LR
  Decl[📝 Declarative<br/>'I want 3 VMs'] --> Tool1[🤖 Tool figures out HOW]
  Imp[🔧 Imperative<br/>'Create VM 1, then VM 2, then VM 3'] --> Tool2[🏃 You define each step]
```

Схема: декларативный подход описывает желаемое состояние («хочу 3 VM»), и инструмент сам решает, как его достичь; в императивном подходе ты задаёшь каждый шаг.

| Aspect | 📝 Declarative | 🔧 Imperative |
|--------|---------------|---------------|
| You write | Desired end state | Sequence of steps |
| Idempotency | Built in — re-run is safe | You implement it |
| Examples | Terraform HCL, CloudFormation, Kubernetes YAML | Bash scripts, AWS SDK calls, Pulumi (mostly) |
| Failure mode | Tool can't figure out a tricky migration | Step 7 fails on re-run |

| Аспект | 📝 Декларативный | 🔧 Императивный |
|--------|------------------|-----------------|
| Что ты пишешь | Желаемое конечное состояние | Последовательность шагов |
| Идемпотентность | Встроена — повторный запуск безопасен | Реализуешь сам |
| Примеры | Terraform HCL, CloudFormation, Kubernetes YAML | Bash-скрипты, вызовы AWS SDK, Pulumi (в основном) |
| Режим отказа | Инструмент не может разобраться в сложной миграции | Шаг 7 падает при повторном запуске |

> 🤔 **Pulumi twist:** code is imperative, but the Pulumi engine resolves it into a *declarative* desired-state graph before talking to the cloud. Best of both worlds.
>
> 🤔 **Нюанс Pulumi:** код императивный, но движок Pulumi превращает его в *декларативный* граф желаемого состояния, прежде чем обратиться к облаку. Лучшее из двух миров.

---

## 📍 Slide 10 – 🛠️ The IaC Tool Landscape (May 2026)

Слайд 10 — Ландшафт инструментов IaC (май 2026)

| Tool | License | Language | Multi-cloud | State |
|------|---------|----------|-------------|-------|
| 🌍 **Terraform 1.15.3** | BSL-1.1 (HashiCorp) | HCL | ✅ 3000+ providers | Local / S3 / TF Cloud |
| 🌱 **OpenTofu 1.12.0** | MPL-2.0 (Linux Foundation) | HCL (Terraform-compatible) | ✅ Same providers | Local / S3 / OCI |
| 📦 **Pulumi 3.243** | Apache-2.0 | Python, TS, Go, C#, Java, YAML | ✅ AWS, GCP, Azure, K8s, … | Pulumi Cloud / S3 / local |
| ☁️ **AWS CloudFormation** | Proprietary | YAML / JSON | ❌ AWS only | Managed by AWS |
| ☁️ **AWS CDK / GCP CDK** | Apache-2.0 | TS, Python, Java, Go | ❌ Per-cloud | Underlying CloudFormation / Deployment Manager |
| ☸️ **Crossplane** | Apache-2.0 | Kubernetes YAML | ✅ via providers | Kubernetes etcd |

| Инструмент | Лицензия | Язык | Multi-cloud | State |
|------------|----------|------|-------------|-------|
| 🌍 **Terraform 1.15.3** | BSL-1.1 (HashiCorp) | HCL | ✅ 3000+ провайдеров | Локально / S3 / TF Cloud |
| 🌱 **OpenTofu 1.12.0** | MPL-2.0 (Linux Foundation) | HCL (совместим с Terraform) | ✅ Те же провайдеры | Локально / S3 / OCI |
| 📦 **Pulumi 3.243** | Apache-2.0 | Python, TS, Go, C#, Java, YAML | ✅ AWS, GCP, Azure, K8s, … | Pulumi Cloud / S3 / локально |
| ☁️ **AWS CloudFormation** | Проприетарная | YAML / JSON | ❌ Только AWS | Управляется AWS |
| ☁️ **AWS CDK / GCP CDK** | Apache-2.0 | TS, Python, Java, Go | ❌ Своя для каждого облака | CloudFormation / Deployment Manager под капотом |
| ☸️ **Crossplane** | Apache-2.0 | Kubernetes YAML | ✅ Через провайдеров | Kubernetes etcd |

> 🔗 **This lecture focuses on Terraform and Pulumi** — the two tools Lab 4 uses. OpenTofu shows up in slide 21.
>
> 🔗 **Эта лекция посвящена Terraform и Pulumi** — именно эти два инструмента используются в лабе 4. OpenTofu появится на слайде 21.

---

## 📍 Slide 11 – 📜 A Brief History

Слайд 11 — Краткая история

* 📅 **July 2014** — Mitchell Hashimoto and Armon Dadgar release **Terraform v0.1** at HashiCorp. First serious multi-cloud IaC tool.
  * 📅 **Июль 2014** — Митчелл Хашимото и Армон Дадгар выпускают **Terraform v0.1** в HashiCorp. Первый серьёзный multi-cloud-инструмент IaC.
* 📅 **September 2019** — **Pulumi v1.0** released by Joe Duffy's team. First IaC tool to embrace general-purpose programming languages.
  * 📅 **Сентябрь 2019** — команда Джо Даффи выпускает **Pulumi v1.0**. Первый инструмент IaC, сделавший ставку на языки общего назначения.
* 📅 **August 10, 2023** — HashiCorp re-licenses Terraform from MPL-2.0 to the **Business Source License (BSL)**. Vendors building products on Terraform suddenly need a paid agreement.
  * 📅 **10 августа 2023** — HashiCorp меняет лицензию Terraform с MPL-2.0 на **Business Source License (BSL)**. Вендорам, строящим продукты на Terraform, внезапно нужен платный договор.
* 📅 **August 2023** — Gruntwork, env0, Spacelift, and Terragrunt fork Terraform 1.5 → **OpenTofu**.
  * 📅 **Август 2023** — Gruntwork, env0, Spacelift и Terragrunt форкают Terraform 1.5 → **OpenTofu**.
* 📅 **September 2023** — OpenTofu is **donated to the Linux Foundation** for vendor-neutral governance; first stable 1.6 release in January 2024.
  * 📅 **Сентябрь 2023** — OpenTofu **передают Linux Foundation** ради управления, независимого от вендоров; первый стабильный релиз 1.6 — в январе 2024.
* 📅 **December 2023** — Linux Foundation announces **OpenBao**, the parallel community fork of Vault (same BSL backstory).
  * 📅 **Декабрь 2023** — Linux Foundation анонсирует **OpenBao**, точно такой же community-форк Vault (та же история с BSL).
* 📅 **May 2026 (now)** — Terraform 1.15.3 and OpenTofu 1.12.0 are CLI-compatible; most enterprises use one or the other, never both.
  * 📅 **Май 2026 (сейчас)** — Terraform 1.15.3 и OpenTofu 1.12.0 совместимы на уровне CLI; большинство предприятий использует один или другой, но не оба.

> 🔥 The BSL change is the single most consequential IaC event of the decade. It's why this lecture treats OpenTofu as first-class.
>
> 🔥 Смена лицензии на BSL — самое значимое событие в мире IaC за десятилетие. Именно поэтому эта лекция относится к OpenTofu как к полноправному инструменту.

---

## 📍 Slide 12 – 🌍 Terraform Workflow

Слайд 12 — Рабочий процесс Terraform

```mermaid
flowchart LR
  Write[📝 Write .tf] --> Init[🔧 init]
  Init --> Plan[📋 plan]
  Plan --> Apply[🚀 apply]
  Apply --> Destroy[💥 destroy]
  Plan -.->|review| Plan
```

Схема: пишешь `.tf`, выполняешь init, затем plan, apply и при необходимости destroy; plan перечитывают перед применением.

```bash
terraform init      # ⬇️ download provider binaries to .terraform/
terraform plan      # 🔍 diff between code, state, and reality
terraform apply     # 🚀 make reality match code
terraform destroy   # 💥 delete everything in this state
```

Пояснение: `init` скачивает бинарники провайдеров в `.terraform/`, `plan` показывает diff между кодом, state и реальностью, `apply` приводит реальность в соответствие с кодом, а `destroy` удаляет всё из этого state.

> ⚠️ **`plan` is sacred.** Read it. Every time. Especially before `apply` in production. CI should post the plan into the PR — Lab 4 bonus wires this up.
>
> ⚠️ **`plan` — это святое.** Читай его. Каждый раз. Особенно перед `apply` в проде. CI должен публиковать plan в PR — бонусное задание лабы 4 это настраивает.

---

## 📍 Slide 13 – 🧱 HCL Building Blocks

Слайд 13 — Строительные блоки HCL

```hcl
# ☁️ provider — which cloud and how to authenticate
provider "aws" {
  region = var.region
}

# 📊 variable — input
variable "region" {
  description = "AWS region"
  type        = string
  default     = "eu-central-1"
}

# 🏗️ resource — something we create and own
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"   # Ubuntu 22.04 LTS, eu-central-1
  instance_type = "t3.micro"
  tags = { Name = "lab4-web", Env = "lab" }
}

# 🔍 data — read-only lookup of something we don't own
data "aws_vpc" "default" { default = true }

# 📤 output — value to surface after apply (and for other modules)
output "public_ip" {
  value = aws_instance.web.public_ip
}
```

Пояснение: `provider` задаёт облако и способ аутентификации, `variable` — входной параметр, `resource` — объект, которым мы управляем, `data` — read-only чтение того, чем мы не владеем, а `output` — значение, которое нужно получить после apply.

> 📝 Five primitives — `provider`, `resource`, `variable`, `output`, `data` — cover **90%** of real Terraform code. Modules are just bundles of these.
>
> 📝 Пять примитивов — `provider`, `resource`, `variable`, `output` и `data` — покрывают **90%** реального кода Terraform. Модули — это просто их наборы.

---

## 📍 Slide 14 – 🔄 `plan` Output — Read It Like A Diff

Слайд 14 — Вывод `plan`: читай его как diff

```text
$ terraform plan

Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t3.micro"
      + public_ip     = (known after apply)
      + tags          = {
          + "Env"  = "lab"
          + "Name" = "lab4-web"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Пояснение: `+` означает создание ресурса; итоговая строка `Plan: 1 to add, 0 to change, 0 to destroy` говорит, что добавится один ресурс, а изменений и удалений нет.

| Symbol | Meaning |
|:------:|---------|
| `+` | Create |
| `-` | Destroy (data loss risk!) |
| `~` | Update in place |
| `-/+` | Destroy and recreate (data loss risk!) |

| Символ | Значение |
|:------:|----------|
| `+` | Создать |
| `-` | Удалить (риск потери данных!) |
| `~` | Обновить на месте |
| `-/+` | Удалить и создать заново (риск потери данных!) |

> 🚨 **`-/+` on a database resource = your data is about to die.** Read every plan, every time, in production.
>
> 🚨 **`-/+` для ресурса базы данных = твои данные вот-вот погибнут.** Читай каждый plan, каждый раз, в проде.

---

## 📍 Slide 15 – 📦 State: The Thing That Makes Terraform Work

Слайд 15 — State: то, на чём держится Terraform

* 🗺️ **State** = a JSON map from your code (`aws_instance.web`) to a real cloud object (`i-0abc123…`)
  * 🗺️ **State** — это JSON-карта из твоего кода (`aws_instance.web`) в реальный объект в облаке (`i-0abc123…`)
* 🔍 Without state, Terraform has no idea what already exists — every apply would try to create everything
  * 🔍 Без state Terraform не знает, что уже существует, — каждый apply пытался бы создать всё заново
* ⚠️ State contains **sensitive data**: passwords, private IPs, sometimes private keys — treat it like a secret
  * ⚠️ В state есть **чувствительные данные**: пароли, приватные IP, иногда приватные ключи — обращайся с ним как с секретом

```mermaid
flowchart LR
  Code[📝 .tf code] --> TF[🌍 terraform]
  State[📦 terraform.tfstate] --> TF
  Cloud[☁️ Real cloud] --> TF
  TF -->|plan diff| You[👀 You]
```

Схема: Terraform сверяет код `.tf`, файл `terraform.tfstate` и реальное облако, а результат сравнения показывает тебе в виде плана.

> ⚠️ **Never edit `terraform.tfstate` by hand.** Use `terraform state` subcommands (`mv`, `rm`, `import`).
>
> ⚠️ **Никогда не редактируй `terraform.tfstate` руками.** Используй подкоманды `terraform state` (`mv`, `rm`, `import`).

---

## 📍 Slide 16 – 🌐 Local vs. Remote State

Слайд 16 — Локальный и удалённый state

| | 📁 Local (`terraform.tfstate` on disk) | 🌐 Remote (S3, GCS, TF Cloud, …) |
|-|----------------------------------------|----------------------------------|
| Solo prototyping | ✅ Fine | Overkill |
| Team of 2+ | ❌ Two engineers apply → race condition, corrupted state | ✅ Locking serializes applies |
| Disaster recovery | ❌ Laptop dies → state lost → cloud orphaned | ✅ Versioned object storage |
| Secrets in state | 🟡 Plaintext on disk | ✅ Encrypted at rest (KMS) |
| CI/CD | ❌ Each runner has stale state | ✅ Shared canonical state |

| | 📁 Локальный (`terraform.tfstate` на диске) | 🌐 Удалённый (S3, GCS, TF Cloud, …) |
|-|----------------------------------------|----------------------------------|
| Соло-прототип | ✅ Нормально | Избыточно |
| Команда из 2+ | ❌ Два инженера делают apply → гонка и испорченный state | ✅ Блокировка сериализует apply |
| Аварийное восстановление | ❌ Умер ноутбук → state потерян → облако осиротело | ✅ Версионируемое объектное хранилище |
| Секреты в state | 🟡 Открытым текстом на диске | ✅ Зашифрован при хранении (KMS) |
| CI/CD | ❌ У каждого раннера устаревший state | ✅ Общий канонический state |

**📦 Remote backend example (S3 + DynamoDB lock):**

**📦 Пример удалённого backend (S3 + блокировка через DynamoDB):**

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-tf-state"
    key            = "lab4/terraform.tfstate"
    region         = "eu-central-1"
    dynamodb_table = "tf-locks"     # 🔒 prevents concurrent applies
    encrypt        = true           # 🔐 SSE-KMS
  }
}
```

Пояснение: backend хранит state в S3, `dynamodb_table` обеспечивает блокировку, чтобы два apply не выполнялись одновременно, а `encrypt = true` включает шифрование SSE-KMS.

> 🔗 **Lab 4 starts with local state on purpose** — so the failure mode of two students applying at once is visible. The bonus task migrates to S3.
>
> 🔗 **Лаба 4 намеренно начинается с локального state** — чтобы был виден режим отказа, когда два студента делают apply одновременно. Бонусное задание переводит state в S3.

---

## 📍 Slide 17 – 🚨 State Pitfalls (Where Students and Pros Both Screw Up)

Слайд 17 — Подводные камни state (где ошибаются и студенты, и профи)

1. 🙅 **Committing `terraform.tfstate` to Git** — it has secrets, and merge conflicts on JSON are unrecoverable. Add it to `.gitignore` on day one.
   * 🙅 **Коммит `terraform.tfstate` в Git** — там секреты, а merge-конфликты в JSON невосстановимы. Добавь его в `.gitignore` в первый же день.
2. 🔓 **No locking on remote state** — two simultaneous `apply`s race; state ends up corrupt; recovery is manual.
   * 🔓 **Нет блокировки удалённого state** — два одновременных `apply` гоняются; state повреждается; восстановление ручное.
3. 📉 **Drift** — someone clicks in the AWS console; Terraform doesn't know; next `apply` reverts or fails.
   * 📉 **Дрейф** — кто-то кликает в консоли AWS; Terraform об этом не знает; следующий `apply` откатывает изменение или падает.
   * Catch it: `terraform plan` in a nightly CI job — any non-zero diff = drift.
     * Поймать это: `terraform plan` в ночной CI-задаче — любой ненулевой diff означает дрейф.
4. 🔄 **Orphaned resources** — you `rm -rf` a module; Terraform forgets about its resources; they bill forever.
   * 🔄 **Осиротевшие ресурсы** — ты делаешь `rm -rf` модуля; Terraform забывает о его ресурсах; они тарифицируются вечно.
   * Use `terraform state list` before deleting code.
     * Используй `terraform state list` перед удалением кода.
5. 📥 **Need to manage existing cloud objects you didn't create with Terraform?**
   * 📥 **Нужно управлять существующими объектами в облаке, которые ты создал не через Terraform?**

```bash
# Adopt an existing EC2 instance into Terraform state
terraform import aws_instance.web i-0abc123def456

# Or, with Terraform 1.5+, declare it once and let TF import on apply
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}
```

Пояснение: `terraform import` добавляет существующий EC2-инстанс в state; начиная с Terraform 1.5 можно объявить блок `import` и позволить Terraform импортировать объект во время apply.

> 🔥 **Drift detection is half the value of IaC.** Cattle that wander off the ranch are still cattle — until somebody renames them.
>
> 🔥 **Детекция дрейфа — половина ценности IaC.** Скот, ушедший с ранчо, остаётся скотом — пока кто-нибудь не переименует его.

---

## 📍 Slide 18 – 📦 Pulumi — Infrastructure in Your Favorite Language

Слайд 18 — Pulumi: инфраструктура на твоём любимом языке

* 🐍 Write infrastructure in **Python, TypeScript, Go, C#, Java**, or YAML
  * 🐍 Пиши инфраструктуру на **Python, TypeScript, Go, C# или Java** — либо на YAML
* 🔧 Loops, conditionals, functions, classes, unit tests — your normal dev tools work
  * 🔧 Циклы, условия, функции, классы, юнит-тесты — привычные инструменты разработчика работают
* 📦 State stored in Pulumi Cloud (free for individuals, paid for teams) or self-hosted (S3, Azure Blob)
  * 📦 State хранится в Pulumi Cloud (бесплатно для индивидуального использования, платно для команд) или в self-hosted-хранилище (S3, Azure Blob)
* 🔐 Secrets **encrypted by default** in state (no opt-in needed)
  * 🔐 Секреты **шифруются по умолчанию** в state (ничего включать отдельно не нужно)

```python
# 🐍 pulumi/__main__.py — same VM as the Terraform example
import pulumi
import pulumi_aws as aws

web = aws.ec2.Instance("web",
    ami="ami-0c55b159cbfafe1f0",
    instance_type="t3.micro",
    tags={"Name": "lab4-web", "Env": "lab"})

pulumi.export("public_ip", web.public_ip)
```

Пояснение: та же самая VM, что и в примере Terraform, описана обычным Python-кодом; `pulumi.export` публикует публичный IP как output.

```bash
pulumi up        # equivalent to terraform plan + apply
pulumi preview   # plan only
pulumi destroy   # tear down
```

Пояснение: `pulumi up` делает то же, что `terraform plan` вместе с `apply`; `pulumi preview` показывает только план, а `pulumi destroy` удаляет ресурсы.

> 🔗 **Lab 4 Task 2** reimplements your Terraform VM in Pulumi Python — you'll feel the difference in 30 minutes.
>
> 🔗 **Задание 2 лабы 4** повторяет твою Terraform-VM на Pulumi Python — разницу подходов почувствуешь за 30 минут.

---

## 📍 Slide 19 – 🧠 Why Code-as-Infrastructure Matters

Слайд 19 — Почему инфраструктура как код так важна

```python
# 🐍 Loops, conditionals, and reuse — natural in Python
for env in ["dev", "staging", "prod"]:
    aws.ec2.Instance(f"web-{env}",
        ami="ami-0c55b159cbfafe1f0",
        instance_type="t3.micro" if env != "prod" else "t3.large",
        tags={"Env": env})
```

Пояснение: в Python цикл по окружениям выглядит естественно — для каждого создаётся инстанс с нужным типом, и прод получает более крупный.

```hcl
# 🌍 The same thing in HCL — works, but the syntax is less natural
locals {
  envs = { dev = "t3.micro", staging = "t3.micro", prod = "t3.large" }
}
resource "aws_instance" "web" {
  for_each      = local.envs
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value
  tags          = { Env = each.key }
}
```

Пояснение: тот же результат на HCL — через `for_each` и `locals`; работает, но синтаксис менее естественный, чем обычный цикл.

> 💡 **The right tool depends on the team.** Python shops with strong dev culture often prefer Pulumi. Platform teams that touch dozens of providers and need HashiCorp's ecosystem usually pick Terraform/OpenTofu. **Lab 4 makes you do both so you can decide for yourself.**
>
> 💡 **Правильный инструмент зависит от команды.** Python-команды с сильной инженерной культурой часто выбирают Pulumi. Платформенные команды, которым нужны десятки провайдеров и экосистема HashiCorp, обычно берут Terraform/OpenTofu. **Лаба 4 заставит тебя сделать и то, и другое, чтобы ты решил сам.**

---

## 📍 Slide 20 – ⚖️ Terraform vs. Pulumi vs. OpenTofu

Слайд 20 — Terraform, Pulumi и OpenTofu

| Aspect | 🌍 Terraform 1.15.3 | 🌱 OpenTofu 1.12.0 | 📦 Pulumi 3.243 |
|--------|--------------------|------------------|------------------|
| 📜 License | BSL-1.1 | MPL-2.0 | Apache-2.0 |
| 🏢 Steward | HashiCorp (IBM, 2025) | Linux Foundation | Pulumi Corp |
| 📝 Language | HCL | HCL (Terraform-compatible) | Python, TS, Go, C#, Java, YAML |
| 🧠 Paradigm | Declarative | Declarative | Imperative → declarative graph |
| 🔄 Logic | `for_each`, `count`, `dynamic` | Same + `for_each` improvements | Full programming language |
| 🧪 Unit testing | Terratest (Go) | Same | Native (pytest, jest, go test) |
| 📦 Default state | Local / S3 / TF Cloud | Local / S3 / OCI | Pulumi Cloud / S3 |
| 🔐 Secrets in state | Plaintext (unless backend encrypts) | Plaintext + native encryption in 1.7+ | Encrypted by default |
| 🌐 Community | Largest (millions of users) | Growing fast (LF-backed) | Smaller but loyal |

| Аспект | 🌍 Terraform 1.15.3 | 🌱 OpenTofu 1.12.0 | 📦 Pulumi 3.243 |
|--------|--------------------|------------------|------------------|
| 📜 Лицензия | BSL-1.1 | MPL-2.0 | Apache-2.0 |
| 🏢 Ведёт | HashiCorp (IBM, 2025) | Linux Foundation | Pulumi Corp |
| 📝 Язык | HCL | HCL (совместим с Terraform) | Python, TS, Go, C#, Java, YAML |
| 🧠 Парадигма | Декларативная | Декларативная | Императивная → декларативный граф |
| 🔄 Логика | `for_each`, `count`, `dynamic` | То же + улучшения `for_each` | Полноценный язык программирования |
| 🧪 Юнит-тесты | Terratest (Go) | То же | Нативные (pytest, jest, go test) |
| 📦 State по умолчанию | Локально / S3 / TF Cloud | Локально / S3 / OCI | Pulumi Cloud / S3 |
| 🔐 Секреты в state | Открытым текстом (если backend не шифрует) | Открытым текстом + встроенное шифрование с 1.7+ | Зашифрованы по умолчанию |
| 🌐 Сообщество | Крупнейшее (миллионы пользователей) | Быстро растёт (при поддержке LF) | Меньше, но лояльное |

> 🎯 **Course default:** Lab 4 accepts any of the three. Use whichever your team uses in real life.
>
> 🎯 **Вариант по умолчанию в курсе:** лаба 4 принимает любой из трёх. Используй тот, что у тебя в реальной команде.

---

## 📍 Slide 21 – 🌱 OpenTofu: Why You Care About the Fork

Слайд 21 — OpenTofu: почему тебе важен этот форк

* 📅 **August 10, 2023** — HashiCorp changes Terraform's license from MPL-2.0 (permissive) to BSL-1.1 (source-available but commercial use restricted)
  * 📅 **10 августа 2023** — HashiCorp меняет лицензию Terraform с MPL-2.0 (разрешительной) на BSL-1.1 (исходники открыты, но коммерческое использование ограничено)
* 😡 Companies whose products *depend on* Terraform (Spacelift, env0, Gruntwork) face existential risk
  * 😡 Компании, чьи продукты *зависят от* Terraform (Spacelift, env0, Gruntwork), оказываются под угрозой существования
* 🍴 **August 2023:** they fork Terraform 1.5 → publish as **OpenTofu** under MPL-2.0
  * 🍴 **Август 2023:** они форкают Terraform 1.5 → и публикуют как **OpenTofu** под MPL-2.0
* 🏛️ **September 2023:** OpenTofu is **donated to the Linux Foundation** → vendor-neutral governance
  * 🏛️ **Сентябрь 2023:** OpenTofu **передаётся Linux Foundation** → управление, независимое от вендоров
* 🏛️ **December 2023:** Linux Foundation announces **OpenBao** — the same fork pattern, applied to Vault
  * 🏛️ **Декабрь 2023:** Linux Foundation анонсирует **OpenBao** — тот же паттерн форка, применённый к Vault
* ✅ **Today:** OpenTofu 1.12.0 is a 1:1 drop-in for Terraform 1.15.3 — same HCL, same providers, same registry (mostly)
  * ✅ **Сегодня:** OpenTofu 1.12.0 — полная замена Terraform 1.15.3 один к одному: тот же HCL, те же провайдеры, тот же registry (в основном)
* 💼 **2025:** IBM acquires HashiCorp; Terraform stays BSL — the fork is permanent
  * 💼 **2025:** IBM приобретает HashiCorp; Terraform остаётся на BSL — форк навсегда

```bash
# Identical workflows
terraform init && terraform plan && terraform apply
tofu init && tofu plan && tofu apply

# Yes, you can copy a Terraform repo and run `tofu` against it
```

Пояснение: команды полностью идентичны — достаточно заменить `terraform` на `tofu`; репозиторий Terraform можно скопировать и запускать через `tofu`.

> 🔥 The right answer in 2026 is *probably* OpenTofu for new projects, Terraform for existing ones with paid HashiCorp support. Pick deliberately.
>
> 🔥 Правильный ответ в 2026-м — *вероятно*, OpenTofu для новых проектов и Terraform для существующих, где есть платная поддержка HashiCorp. Выбирай осознанно.

---

## 📍 Slide 22 – 🔐 Security: Don't Be a Headline

Слайд 22 — Безопасность: не попади в заголовки новостей

**Five rules. Memorize them.**

**Пять правил. Запомни их.**

1. 🚫 **Never commit credentials.** Not in `.tf`, not in `.tfvars`, not in `provider {}` blocks.
   * 🚫 **Никогда не коммить учётные данные.** Ни в `.tf`, ни в `.tfvars`, ни в блоках `provider {}`.

```hcl
# ❌ NEVER
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# ✅ Let the provider read env vars or AWS profile
provider "aws" {
  region = var.region
}
```

Пояснение: первый вариант — ключи прямо в коде, так делать нельзя; во втором провайдер берёт учётные данные из переменных окружения или AWS-профиля.

2. 🙈 **Never commit `terraform.tfstate` or `*.tfstate.backup`** — they contain decrypted secrets. Add to `.gitignore` line 1.
   * 🙈 **Никогда не коммить `terraform.tfstate` или `*.tfstate.backup`** — там расшифрованные секреты. Добавь их в `.gitignore` первой строкой.
3. 🌐 **Use a remote backend with encryption + locking** — S3 + SSE-KMS + DynamoDB lock, GCS with versioning, or Pulumi Cloud.
   * 🌐 **Используй удалённый backend с шифрованием и блокировками** — S3 + SSE-KMS + блокировка в DynamoDB, GCS с версионированием или Pulumi Cloud.
4. 🔑 **Use OIDC for CI cloud auth, not long-lived keys.** GitHub Actions → AWS / GCP via `aws-actions/configure-aws-credentials` or `google-github-actions/auth` — no secrets in repo, ever.
   * 🔑 **Для аутентификации CI в облаке используй OIDC, а не долгоживущие ключи.** GitHub Actions → AWS / GCP через `aws-actions/configure-aws-credentials` или `google-github-actions/auth` — никаких секретов в репозитории, никогда.
5. 🔍 **Scan IaC code.** `tfsec`, `checkov`, `terrascan` — wire them into CI. Catches public S3 buckets, missing encryption, IAM wildcards.
   * 🔍 **Сканируй код IaC.** `tfsec`, `checkov`, `terrascan` — встрой их в CI. Они ловят публичные S3-бакеты, отсутствие шифрования и wildcard в IAM.

> 🔗 **Course tie-in:** Lab 4 bonus adds a `terraform plan` + `checkov` step on every PR. DevSecOps elective goes deeper.
>
> 🔗 **Связь с курсом:** бонус лабы 4 добавляет шаг `terraform plan` + `checkov` на каждый PR. Электив DevSecOps идёт глубже.

---

## 📍 Slide 23 – 💥 Real Incidents — Why These Rules Exist

Слайд 23 — Реальные инциденты: зачем нужны эти правила

* 🏦 **Code Spaces (2014)** — credentials in a repo → AWS console takeover → attacker deleted S3 + EC2 + RDS → company *literally shut down* in 12 hours. The textbook "don't commit credentials" story.
  * 🏦 **Code Spaces (2014)** — учётные данные в репозитории → захват консоли AWS → атакующий удалил S3, EC2 и RDS → компания *буквально закрылась* за 12 часов. Хрестоматийная история «не коммить учётные данные».
* 🚗 **Uber (2016)** — AWS keys hardcoded in a private GitHub repo → 57M user records exfiltrated → $148M settlement. Private ≠ safe.
  * 🚗 **Uber (2016)** — ключи AWS были захардкожены в приватном репозитории GitHub → 57 млн записей пользователей утекли → $148 млн мирового соглашения. «Приватный» ≠ «безопасный».
* 🏭 **2024 SAP Concur** — internal Terraform state in an unencrypted S3 bucket leaked employee PII for years.
  * 🏭 **SAP Concur, 2024** — внутренний Terraform state в незашифрованном S3-бакете годами сливал персональные данные сотрудников.
* 🐍 **2025 SnakeKeylogger / Codefinger** — attackers found leaked Pulumi Cloud tokens in public repos and used them to enumerate and ransom S3 buckets via SSE-C.
  * 🐍 **SnakeKeylogger / Codefinger, 2025** — атакующие нашли утёкшие токены Pulumi Cloud в публичных репозиториях и использовали их, чтобы перечислить S3-бакеты и потребовать выкуп через SSE-C.
* 📉 **Knight Capital (2012, pre-IaC)** — manual deploy to 7 of 8 servers, the 8th still had old code → $440M loss in 45 minutes. Manual = drift = catastrophe.
  * 📉 **Knight Capital (2012, до IaC)** — ручной деплой на 7 из 8 серверов, на 8-м остался старый код → $440 млн потерь за 45 минут. Ручные действия = дрейф = катастрофа.

> 💡 **Every one of these would have been prevented by following the rules on slide 22.** The rules are paid for in real money.
>
> 💡 **Каждого из этих инцидентов можно было избежать, следуя правилам со слайда 22.** За эти правила заплачено реальными деньгами.

---

## 📍 Slide 24 – 🎯 Key Takeaways

Слайд 24 — Ключевые выводы

1. 🏗️ **Infrastructure as Code = your cloud lives in Git** — reproducible, disposable, consistent, visible
   * 🏗️ **Инфраструктура как код = твоё облако живёт в Git** — воспроизводимо, одноразово, согласованно, прозрачно
2. 🐄 **Cattle, not pets** — name servers by role + number, never as individuals
   * 🐄 **Скот, а не питомцы** — называй серверы по роли и номеру, никогда как индивидуальностей
3. 🔄 **Read the `plan`** — every time, especially before production apply
   * 🔄 **Читай `plan`** — каждый раз, особенно перед apply в прод
4. 📦 **Remote, locked, encrypted state from day one** in any team setting
   * 📦 **Удалённый, заблокированный и зашифрованный state с первого дня** в любой командной работе
5. 🌍 **Terraform/OpenTofu** for big ecosystems and declarative simplicity; **Pulumi** for code-heavy teams and complex logic
   * 🌍 **Terraform/OpenTofu** — для больших экосистем и декларативной простоты; **Pulumi** — для команд с сильной культурой кода и сложной логикой
6. 🌱 **OpenTofu exists because licensing matters** — know the story, pick deliberately
   * 🌱 **OpenTofu существует, потому что лицензии важны** — знай историю и выбирай осознанно
7. 🔐 **No secrets in code. No state in Git. OIDC for CI.** Three rules; non-negotiable.
   * 🔐 **Никаких секретов в коде. Никакого state в Git. OIDC для CI.** Три правила; без исключений.

> 💡 **If you can't `git clone && terraform apply` from scratch, it's not IaC — it's a haunted spreadsheet of clicks.**
>
> 💡 **Если ты не можешь выполнить `git clone && terraform apply` с нуля, это не IaC — это проклятая таблица с кликами.**

---

## 📍 Slide 25 – 🧠 The Mindset Shift

Слайд 25 — Смена мышления

| 😰 Old | 🚀 IaC-native |
|--------|--------------|
| 🙅 "SSH and fix it" | 📝 "Change the code, plan, apply" |
| 🚫 "Don't touch web-03" | 💪 "Destroy web-03 and let TF rebuild it" |
| 👉 "Who set this up?" | 📜 `git blame` answers it |
| 😨 "Manual is faster" | ⚡ "Manual is faster *once*. IaC is faster forever." |
| 💻 "Works on my cloud" | 🌍 "Works in any account that runs the code" |
| 🤷 "We'll document it later" | 📝 "The code IS the documentation" |

| 😰 Старый взгляд | 🚀 IaC-подход |
|------------------|---------------|
| 🙅 «Зайти по SSH и починить» | 📝 «Изменить код, посмотреть plan, применить» |
| 🚫 «Не трогай web-03» | 💪 «Уничтожь web-03 и дай TF пересобрать его» |
| 👉 «Кто это настроил?» | 📜 `git blame` ответит |
| 😨 «Руками быстрее» | ⚡ «Руками быстрее *один раз*. IaC быстрее всегда» |
| 💻 «Работает в моём облаке» | 🌍 «Работает в любом аккаунте, где запущен этот код» |
| 🤷 «Задокументируем потом» | 📝 «Код И ЕСТЬ документация» |

> ❓ Which column describes the team you want to work on?
>
> ❓ Какая колонка описывает команду, в которой ты хочешь работать?

---

## 📍 Slide 26 – 🚀 What Comes Next

Слайд 26 — Что дальше

**📚 Next lecture: *Configuration Management with Ansible.*** Terraform builds the VM; Ansible configures what runs *inside* it. The two complete each other.

**📚 Следующая лекция: *Управление конфигурацией с Ansible.*** Terraform строит VM, а Ansible настраивает то, что работает *внутри* неё. Они дополняют друг друга.

* 🔧 Idempotent playbooks over SSH
  * 🔧 Идемпотентные playbook-и по SSH
* 📦 Roles, inventories, and Ansible Galaxy
  * 📦 Роли, инвентари и Ansible Galaxy
* 🤖 Provisioning Docker on the VMs you'll create in Lab 4
  * 🤖 Развёртывание Docker на VM, которые ты создашь в лабе 4

**🔬 Lab 4 this week:** create a cloud VM with both Terraform *and* Pulumi, compare the experience, and (bonus) wire `terraform plan` into a GitHub Actions PR check. Keep the VM running — **Lab 5 will SSH into it**.

**🔬 Лаба 4 на этой неделе:** создай облачную VM и через Terraform, *и* через Pulumi, сравни опыт и (бонусно) встрой `terraform plan` в проверку PR в GitHub Actions. Оставь VM работающей — **лаба 5 подключится к ней по SSH**.

```mermaid
flowchart LR
  Lab4[🏗️ Lab 4: TF + Pulumi VM] --> Lab5[🔧 Lab 5: Ansible configures it]
  Lab5 --> Lab9[☸️ Lab 9: K8s on similar VMs]
  Lab9 --> Future[🚀 Labs 13+: GitOps on top]
```

Схема: лаба 4 создаёт VM, лаба 5 настраивает её через Ansible, лаба 9 поднимает Kubernetes на похожих VM, а начиная с лабы 13 сверху появляется GitOps.

**👋 See you in Lecture 5.**

**👋 Увидимся на лекции 5.**

> 🐄 From snowflakes to cattle — one `terraform apply` at a time.
>
> 🐄 От уникальных серверов к скоту — по одному `terraform apply` за раз.

---

## 📚 Resources & Further Reading

📚 Материалы и дополнительное чтение

**📕 Books:**

**📕 Книги:**

* *Terraform: Up & Running* — Yevgeniy Brikman, O'Reilly (3rd ed., 2022; 4th ed. covers OpenTofu, late 2025)
  * *Terraform: Up & Running* — Евгений Брикман, O'Reilly (3-е изд., 2022; 4-е изд. про OpenTofu — конец 2025)
* *Infrastructure as Code* — Kief Morris, O'Reilly (2nd ed., 2020). The reference text.
  * *Infrastructure as Code* — Киф Моррис, O'Reilly (2-е изд., 2020). Эталонная книга.
* *Pulumi: Up & Running* — Adam Gordon Bell, O'Reilly (2024).
  * *Pulumi: Up & Running* — Адам Гордон Белл, O'Reilly (2024).
* *The Phoenix Project* — Kim, Behr, Spafford. Still the best dramatization of the problems IaC solves.
  * *The Phoenix Project* — Ким, Бер, Спаффорд. По-прежнему лучшая драматизация проблем, которые решает IaC.

**🔗 Links:**

**🔗 Ссылки:**

* 🌐 [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform) — official Terraform docs (1.10)
  * 🌐 [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform) — официальная документация Terraform (1.10)
* 🌐 [opentofu.org](https://opentofu.org) — OpenTofu docs and migration guide
  * 🌐 [opentofu.org](https://opentofu.org) — документация OpenTofu и руководство по миграции
* 🌐 [pulumi.com/docs](https://www.pulumi.com/docs/) — official Pulumi docs (3.140)
  * 🌐 [pulumi.com/docs](https://www.pulumi.com/docs/) — официальная документация Pulumi (3.140)
* 🌐 [registry.terraform.io](https://registry.terraform.io/) — providers + community modules (works for OpenTofu too)
  * 🌐 [registry.terraform.io](https://registry.terraform.io/) — провайдеры и community-модули (работает и для OpenTofu)
* 🌐 [github.com/aquasecurity/tfsec](https://github.com/aquasecurity/tfsec) — IaC security scanner
  * 🌐 [github.com/aquasecurity/tfsec](https://github.com/aquasecurity/tfsec) — сканер безопасности IaC

**🎓 Quiz:** post-lecture quiz feeds the weeks 4–6 leaderboard window.

**🎓 Квиз:** квиз после лекции идёт в зачёт недель 4–6.
