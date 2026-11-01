# Lab 5 — Ansible Fundamentals

Лаба 5 — основы Ansible

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Configuration%20Management-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Ansible--core%202.21-informational)

> **Goal:** Configure the VM you provisioned in Lab 4 by writing three reusable Ansible roles (`common`, `docker`, `app_deploy`), then prove the whole thing is idempotent — second run, `changed=0`.
>
> **Цель:** настроить VM, которую ты подготовил в лабе 4, написав три переиспользуемые роли Ansible (`common`, `docker`, `app_deploy`), и затем доказать, что всё идемпотентно — второй запуск даёт `changed=0`.
>
> **Deliverable:** A PR from `lab05` with the `ansible/` project (three roles, two playbooks, encrypted Vault file, docs).
>
> **Результат:** PR из ветки `lab05` с проектом `ansible/` (три роли, два плейбука, зашифрованный Vault-файл, документация).

---

## Overview

Обзор

In this lab you will practice:

В этой лабе ты попрактикуешься в следующем:

- Writing **idempotent** tasks with FQCN modules (`ansible.builtin.*`, `community.docker.*`)
  * Писать **идемпотентные** задачи с модулями по FQCN (`ansible.builtin.*`, `community.docker.*`)
- Organising tasks into **roles** (`tasks/`, `handlers/`, `defaults/`, `templates/`) instead of one giant playbook
  * Организовывать задачи в **роли** (`tasks/`, `handlers/`, `defaults/`, `templates/`), а не в один огромный плейбук
- Using **handlers** so a service restarts only when its config actually changed
  * Использовать **handlers**, чтобы сервис перезапускался только при реальном изменении конфигурации
- Encrypting secrets with **Ansible Vault** so they live in Git safely
  * Шифровать секреты с помощью **Ansible Vault**, чтобы их можно было безопасно хранить в Git
- Reading PLAY RECAP output: `changed=N` on run 1, `changed=0` on run 2 — the acceptance test of configuration management
  * Читать вывод PLAY RECAP: `changed=N` при первом запуске и `changed=0` при втором — это и есть приёмочный тест управления конфигурацией

> ⚠️ **Scope:** Lab 4 *provisioned* the box (Terraform/Pulumi gave it an IP). Lab 5 *configures* what's inside. Lab 6 will add tags, blocks, Compose, and a CI workflow on top of these roles — and will **rename `app_deploy` → `web_app`** via `git mv`. The role names you pick this week will not be permanent for `app_deploy`; everything else carries through unchanged.
>
> ⚠️ **Рамки:** лаба 4 *подготовила* сервер (Terraform/Pulumi выдали ему IP). Лаба 5 *настраивает* то, что внутри. Лаба 6 добавит теги, блоки, Compose и CI-воркфлоу поверх этих ролей — и через `git mv` **переименует `app_deploy` → `web_app`**. Имя роли `app_deploy` не будет окончательным; всё остальное перейдёт без изменений.

> 💡 **No cloud VM? You can target localhost** — set `ansible_connection: local` in the inventory and skip `become:` on tasks that need root in a sudo-less sandbox. The role *logic* is what's graded; running cleanly against localhost with `--check` (where `become` is the only blocker) is accepted as a deliverable. Capture both PLAY RECAPs the same way (`changed=N` then `changed=0`).
>
> 💡 **Нет облачной VM? Можно нацелиться на localhost** — укажи `ansible_connection: local` в inventory и убери `become:` в задачах, которым нужен root в песочнице без sudo. Оценивается именно *логика* ролей; чистый прогон на localhost с `--check` (где `become` — единственное препятствие) принимается как результат. Зафиксируй оба PLAY RECAP так же (`changed=N`, затем `changed=0`).

> **A note on "LTS":** `ansible-core` has **no formal LTS label**. The open-source engine commits to ~6 months of security fixes per minor release; currently supported versions are **2.18, 2.19, 2.20, 2.21**. Use **2.21**. Red Hat's commercial *Ansible Automation Platform* offers extended support — that is a different product. Slides on the internet that say "2.17 LTS" predate the policy clarification.
>
> **Про «LTS»:** у `ansible-core` **нет официального статуса LTS**. Open-source-движок обещает около 6 месяцев security-фиксов на каждый минорный релиз; сейчас поддерживаются версии **2.18, 2.19, 2.20, 2.21**. Используй **2.21**. Коммерческая *Ansible Automation Platform* от Red Hat предлагает расширенную поддержку — это другой продукт. Слайды в интернете с надписью «2.17 LTS» появились до разъяснения политики.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно быть готово из предыдущих лаб:**

- Lab 2: a Docker image of your Python service pushed to Docker Hub (or any registry)
  * Лаба 2: Docker-образ твоего Python-сервиса, залитый в Docker Hub (или любой другой registry)
- Lab 4: a reachable Ubuntu VM (cloud or local libvirt) with passwordless `sudo` for your login user
  * Лаба 4: доступная Ubuntu VM (в облаке или локальный libvirt) с безпарольным `sudo` для твоего пользователя входа

**This lab adds:**

**Что добавляет эта лаба:**

- `ansible/` — three roles, two playbooks, static inventory, encrypted Vault, docs
  * `ansible/` — три роли, два плейбука, статический inventory, зашифрованный Vault, документация

By Lab 6 these same roles get tags + blocks + a CI workflow; by Lab 9 the same image runs on Kubernetes. The decisions you make this week — role granularity, variable layout, FQCN — outlive Lab 5.

К лабе 6 эти же роли получат теги, блоки и CI-воркфлоу; к лабе 9 тот же образ будет работать в Kubernetes. Решения, которые ты примешь на этой неделе — гранулярность ролей, раскладка переменных, FQCN — переживут лабу 5.

---

## Setup

Подготовка

Control node (your laptop or CI runner):

Control node (твой ноутбук или CI-раннер):

```bash
python3 --version          # 3.11+
python3 -m venv .venv && source .venv/bin/activate
pip install "ansible-core==2.21.*"
ansible --version          # expect: ansible [core 2.21.x] / python 3.11+
ansible-galaxy collection install community.docker community.general
```

Пояснение: создаём виртуальное окружение и ставим `ansible-core` 2.21, затем ставим коллекции `community.docker` и `community.general` — они понадобятся для ролей.

Target VM (from Lab 4):

Целевая VM (из лабы 4):

- Ubuntu 24.04 LTS (or 22.04)
  * Ubuntu 24.04 LTS (или 22.04)
- SSH access with your key
  * SSH-доступ по твоему ключу
- **Passwordless `sudo`** for the login user (cloud-init's default — `become:` fails without it)
  * **Безпарольный `sudo`** для пользователя входа (по умолчанию это делает cloud-init — без этого `become:` падает)
- Python 3 present (default on Ubuntu)
  * Python 3 (по умолчанию есть в Ubuntu)

Create the directory layout (you'll fill the files yourself):

Создай структуру каталогов (файлы заполнишь сам):

```
ansible/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all/
│       └── vault.yml             # encrypted; created in Task 3
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml
│   │   └── defaults/main.yml
│   ├── docker/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── defaults/main.yml
│   │   └── templates/daemon.json.j2
│   └── app_deploy/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── defaults/main.yml
├── playbooks/
│   ├── provision.yml             # applies common + docker
│   └── deploy.yml                # applies app_deploy
└── docs/
    └── LAB05.md                  # your submission report
```

Пояснение: это целевая структура проекта `ansible/` — inventory, group_vars, три роли, два плейбука и каталог docs.

> Only create the subdirectories you actually use. A role with no handlers needs no `handlers/`.
>
> Создавай только те подкаталоги, которые реально используешь. Роли без handlers не нужен каталог `handlers/`.

---

## Task 1 — Inventory & Role Scaffolding (2 pts)

Задание 1 — Inventory и каркас ролей (2 балла)

### 1.1 — Why FQCN, and why now

1.1 — Зачем FQCN и почему сейчас

Since the 2021 `ansible-core` / `ansible` split, modules are referenced by their **Fully-Qualified Collection Name**: `ansible.builtin.apt`, not bare `apt`; `community.docker.docker_container`, not bare `docker_container`. Two reasons it matters:

После разделения `ansible-core` / `ansible` в 2021 году модули указывают по **Fully-Qualified Collection Name**: `ansible.builtin.apt`, а не просто `apt`; `community.docker.docker_container`, а не просто `docker_container`. Это важно по двум причинам:

1. **Disambiguation.** Two collections can ship a module with the same short name. Bare names silently resolve to whichever loaded first — a Heisenbug waiting to happen on a CI runner with a different collection set.
   * **Устранение неоднозначности.** Две коллекции могут поставлять модуль с одинаковым коротким именем. Короткие имена молча разрешаются в тот модуль, который загрузился первым, — это «гейзенбаг», который ждёт своего часа на CI-раннере с другим набором коллекций.
2. **Future-proofing.** The short-name shortcut is deprecated; new modules ship FQCN-only. Lab 6's CI workflow will lint your roles for bare names.
   * **Защита на будущее.** Сокращение до короткого имени устарело; новые модули выходят только с FQCN. CI-воркфлоу лабы 6 будет проверять твои роли на короткие имена.

Bare names "work" in 2026 the way `from __future__ import ...` "worked" in Python 2 — until the day they don't. Use FQCN everywhere.

Короткие имена «работают» в 2026 году так же, как `from __future__ import ...` «работал» в Python 2, — до того дня, когда перестанут. Используй FQCN везде.

### 1.2 — Static inventory

1.2 — Статический inventory

`YOUR TASK`: write `inventory/hosts.ini` that puts your Lab 4 VM in a `webservers` group. Required keys per host: `ansible_host` (the IP), `ansible_user` (`ubuntu`), and a `[webservers:vars]` block with `ansible_ssh_private_key_file` (path to your Lab 4 key) and `ansible_python_interpreter=/usr/bin/python3`.

`YOUR TASK`: напиши `inventory/hosts.ini`, который помещает VM из лабы 4 в группу `webservers`. Обязательные ключи для каждого хоста: `ansible_host` (IP), `ansible_user` (`ubuntu`), а также блок `[webservers:vars]` с `ansible_ssh_private_key_file` (путь к ключу из лабы 4) и `ansible_python_interpreter=/usr/bin/python3`.

Skeleton (fill the blanks; everything in `___` is yours):

Скелет (заполни пропуски; всё, что в `___`, — твоё):

```ini
[webservers]
lab-vm ansible_host=___ ansible_user=___        # YOUR TASK: VM IP + SSH user (Lab 4: ubuntu)

[webservers:vars]
ansible_ssh_private_key_file=___                 # YOUR TASK: path to your Lab 4 private key (Lab 4 default: ~/.ssh/lab04)
ansible_python_interpreter=___                   # YOUR TASK: usually /usr/bin/python3
```

<details>
<summary>💡 Targeting localhost instead (sudo-less sandbox)</summary>

💡 Нацеливание на localhost (песочница без sudo)

If you don't have a Lab 4 VM handy and want to run on the control node itself:

Если у тебя нет под рукой VM из лабы 4 и ты хочешь запускать на самом control node:

```ini
[webservers]
lab-vm ansible_host=localhost ansible_user=___ ansible_connection=local   # local exec, don't SSH

[webservers:vars]
ansible_python_interpreter=/usr/bin/python3
```

`ansible_connection: local` makes Ansible skip SSH entirely and `exec` modules in-process. The same roles run; tasks needing root (`apt`, `service`) require local `sudo`. If your sandbox has no passwordless sudo, accept that those tasks won't apply for real but their *logic* still validates in `--check` mode — note that explicitly in your `docs/LAB05.md` and capture the `--check` PLAY RECAPs as your idempotency proof.

Режим `ansible_connection: local` заставляет Ansible полностью пропустить SSH и выполнять модули в том же процессе через `exec`. Те же роли работают; задачи, требующие root (`apt`, `service`), нуждаются в локальном `sudo`. Если в песочнице нет безпарольного sudo, прими, что эти задачи не применятся по-настоящему, но их *логика* всё равно проверится в режиме `--check` — явно отметь это в `docs/LAB05.md` и приложи PLAY RECAP из `--check` как доказательство идемпотентности.

</details>

### 1.3 — `ansible.cfg`

1.3 — `ansible.cfg`

`YOUR TASK`: write `ansible.cfg` with these keys (look up the exact section headers in the docs — don't copy from Stack Overflow, half the answers there set `host_key_checking=False` globally which is a lab convenience, not a prod practice):

`YOUR TASK`: напиши `ansible.cfg` с такими ключами (точные заголовки секций ищи в документации — не копируй со Stack Overflow: половина ответов там глобально выставляет `host_key_checking=False`, это удобство для лабы, а не практика для прода):

- `[defaults]`: `inventory` (path to your inventory), `roles_path` (path to your roles dir), `host_key_checking` (False for lab; understand why prod sets it to True), `retry_files_enabled` (False)
  * `[defaults]`: `inventory` (путь к твоему inventory), `roles_path` (путь к каталогу ролей), `host_key_checking` (False для лабы; пойми, почему в проде его ставят в True), `retry_files_enabled` (False)
- `[privilege_escalation]`: `become`, `become_method`, `become_user`
  * `[privilege_escalation]`: `become`, `become_method`, `become_user`

```ini
[defaults]
inventory = ___                                   # YOUR TASK
roles_path = ___                                  # YOUR TASK
host_key_checking = ___                           # YOUR TASK (lab: False; explain why prod is True)
retry_files_enabled = ___                         # YOUR TASK

[privilege_escalation]
become = ___                                      # YOUR TASK
become_method = ___                               # YOUR TASK
become_user = ___                                 # YOUR TASK
```

Пояснение: `inventory` и `roles_path` задают, откуда Ansible берёт хосты и роли, а секция `[privilege_escalation]` — как выполняется повышение прав.

### 1.4 — Test connectivity

1.4 — Проверка связности

```bash
cd ansible/
ansible all -m ansible.builtin.ping
ansible webservers -a "uname -a"
```

The ping must return `pong` (illustrative — your hostname will differ):

Ping должен вернуть `pong` (пример; у тебя имя хоста будет другим):

```text
lab-vm | SUCCESS => { "changed": false, "ping": "pong" }
```

### 1.5 — Proof of work

1.5 — Подтверждение работы

Paste into `docs/LAB05.md`:

Вставь в `docs/LAB05.md`:

- `ansible --version` output (must show 2.21.x)
  * вывод `ansible --version` (должен показать 2.21.x)
- `ansible all -m ansible.builtin.ping` output (the `pong`)
  * вывод `ansible all -m ansible.builtin.ping` (тот самый `pong`)
- `ansible webservers -a "uname -a"` output (proves SSH + shell access)
  * вывод `ansible webservers -a "uname -a"` (доказывает SSH-доступ и доступ к shell)
- Two sentences on **why FQCN matters** (in your own words)
  * два предложения о том, **почему важен FQCN** (своими словами)

---

## Task 2 — Provisioning Roles: `common` + `docker` (3 pts)

Задание 2 — роли подготовки: `common` + `docker` (3 балла)

A *role* is a directory tree with conventional subfolders. Drop one into a playbook with one line. The rule: if you'd copy-paste the same tasks between two playbooks, it's a role.

*Роль* — это дерево каталогов с привычными подпапками. Подключается к плейбуку одной строкой. Правило: если ты готов копировать одни и те же задачи между двумя плейбуками, это роль.

### 2.1 — `common` role — base packages + timezone

2.1 — роль `common` — базовые пакеты и часовой пояс

`YOUR TASK`: write `roles/common/defaults/main.yml` and `roles/common/tasks/main.yml`. The defaults file holds *variables a user might want to override*; the tasks file holds the actual work.

`YOUR TASK`: напиши `roles/common/defaults/main.yml` и `roles/common/tasks/main.yml`. Файл defaults содержит *переменные, которые пользователь может переопределить*; файл tasks содержит саму работу.

`roles/common/defaults/main.yml`:

```yaml
---
common_packages:                                  # YOUR TASK: list at least 4 base packages
  - ___                                           #   (curl/git/vim/htop/ca-certificates — pick a sensible set)
  - ___
  - ___
  - ___
common_timezone: ___                              # YOUR TASK: Etc/UTC or your region
```

`roles/common/tasks/main.yml` — module names given (those are *the lecture's choice*); args are yours:

`roles/common/tasks/main.yml` — имена модулей даны (это *выбор лекции*); аргументы — твои:

```yaml
---
- name: ___                                       # YOUR TASK: one-line action description
  ansible.builtin.apt:                            # module given
    update_cache: ___                             # YOUR TASK
    cache_valid_time: ___                         # YOUR TASK: why a value > 0? read the apt module's doc
  when: ___                                       # YOUR TASK: limit to Debian-family hosts (hint: ansible_os_family)

- name: ___                                       # YOUR TASK
  ansible.builtin.package:                        # module given — note: NOT `apt:`; see Common Pitfalls
    name: ___                                     # YOUR TASK: reference the variable from defaults
    state: ___                                    # YOUR TASK: present? latest? justify your choice in docs/LAB05.md
  become: ___                                     # YOUR TASK: true or false? why?
  tags: [___]                                     # YOUR TASK: pick a tag — Lab 6 will leverage tag taxonomy

- name: ___                                       # YOUR TASK: set system timezone
  community.general.timezone:                     # module given
    name: ___                                     # YOUR TASK: reference common_timezone
  become: ___                                     # YOUR TASK
  tags: [___]                                     # YOUR TASK
```

Пояснение: в этом скелете тебе нужно заполнить `name`, аргументы модулей и теги; `ansible.builtin.package` выбран вместо `apt`, чтобы роль работала не только на Debian-семействе.

A few of the *whys* you must answer in `docs/LAB05.md` (one sentence each):

Несколько вопросов «почему», на которые ты обязан ответить в `docs/LAB05.md` (по одному предложению на каждый):

- **`become: true` is selective, not blanket.** Why does the timezone task need root but a `ping` doesn't? Why is `become: true` on the *play* a code smell when a single task is the only privilege escalator? (Hint: blast radius + readability.)
  * **`become: true` применяется точечно, а не ко всему подряд.** Почему задаче с часовым поясом нужен root, а `ping` — нет? Почему `become: true` на уровне *play* — признак плохого кода, если единственный потребитель привилегий — одна задача? (Подсказка: радиус поражения и читаемость.)
- **`state: present` vs `state: latest`.** Both install the package. Why does `present` produce reproducible runs and `latest` quietly break idempotency the moment upstream publishes a new version?
  * **`state: present` против `state: latest`.** Оба варианта устанавливают пакет. Почему `present` даёт воспроизводимые прогоны, а `latest` тихо ломает идемпотентность в тот момент, когда upstream публикует новую версию?
- **Why tag tasks now?** Lab 6 will run `ansible-playbook --tags docker` to deploy only the Docker bits. If you don't tag now, Lab 6's first step is "go back and tag everything." Build the habit.
  * **Зачем ставить теги уже сейчас?** Лаба 6 будет запускать `ansible-playbook --tags docker`, чтобы развернуть только Docker-часть. Если не проставить теги сейчас, первый шаг лабы 6 — «вернись и проставь теги везде». Вырабатывай привычку.

### 2.2 — `docker` role — install Docker engine

2.2 — роль `docker` — установка Docker Engine

`roles/docker/defaults/main.yml`:

```yaml
---
docker_user: ___                                  # YOUR TASK: the login user to add to the docker group
docker_log_max_size: ___                          # YOUR TASK: a sane value, e.g. "10m"
```

`roles/docker/templates/daemon.json.j2` — Jinja2 template rendered to `/etc/docker/daemon.json`:

`roles/docker/templates/daemon.json.j2` — шаблон Jinja2, который рендерится в `/etc/docker/daemon.json`:

```jinja
{
  "log-driver": "___",                            # YOUR TASK: choose a driver (json-file or local)
  "log-opts": { "max-size": "{{ ___ }}" }         # YOUR TASK: reference your default
}
```

`roles/docker/handlers/main.yml`:

```yaml
---
- name: ___                                       # YOUR TASK: handler name — keep it short, you'll `notify:` it
  ansible.builtin.service:                        # module given
    name: ___                                     # YOUR TASK
    state: ___                                    # YOUR TASK: restarted? reloaded? why?
```

`roles/docker/tasks/main.yml` — the heavy lifting. Module names are given (those follow the lecture's slide-20 example); the args are the skill:

`roles/docker/tasks/main.yml` — основная работа. Имена модулей даны (они следуют примеру со слайда 20 лекции); аргументы — это и есть навык:

```yaml
---
- name: ___                                       # YOUR TASK: install prerequisites
  ansible.builtin.apt:                            # module given
    name: ___                                     # YOUR TASK: list — apt-transport-https, ca-certificates, curl, gnupg, lsb-release
    state: ___                                    # YOUR TASK
    update_cache: ___                             # YOUR TASK

- name: ___                                       # YOUR TASK: add Docker's official APT key
  ansible.builtin.get_url:                        # module given (deb822_repository is also valid — pick one and justify)
    url: ___                                      # YOUR TASK: the Docker download URL for the gpg key
    dest: ___                                     # YOUR TASK: a path under /etc/apt/keyrings/
    mode: ___                                     # YOUR TASK: file mode (string, e.g. "0644")

- name: ___                                       # YOUR TASK: add the Docker APT repository
  ansible.builtin.apt_repository:                 # module given
    repo: ___                                     # YOUR TASK — use {{ ansible_distribution_release }} so it works on jammy + noble
    state: ___                                    # YOUR TASK
    filename: ___                                 # YOUR TASK: a stable filename in /etc/apt/sources.list.d/

- name: ___                                       # YOUR TASK: install docker-ce + cli + containerd.io
  ansible.builtin.apt:                            # module given
    name: ___                                     # YOUR TASK: list of three packages
    state: ___                                    # YOUR TASK
    update_cache: ___                             # YOUR TASK
  notify: ___                                     # YOUR TASK: name your handler from above

- name: ___                                       # YOUR TASK: render daemon.json
  ansible.builtin.template:                       # module given
    src: ___                                      # YOUR TASK: filename in templates/
    dest: ___                                     # YOUR TASK: /etc/docker/daemon.json
    mode: ___                                     # YOUR TASK
  notify: ___                                     # YOUR TASK: fires ONLY if the rendered file differs

- name: ___                                       # YOUR TASK: ensure docker service is started + enabled
  ansible.builtin.service:                        # module given
    name: ___                                     # YOUR TASK
    state: ___                                    # YOUR TASK
    enabled: ___                                  # YOUR TASK

- name: ___                                       # YOUR TASK: add the user to the docker group
  ansible.builtin.user:                           # module given
    name: ___                                     # YOUR TASK: {{ docker_user }}
    groups: ___                                   # YOUR TASK
    append: ___                                   # YOUR TASK: true — why is `append` critical here? what does the default do?

- name: ___                                       # YOUR TASK: install the Docker SDK for Python
  ansible.builtin.pip:                            # module given — without this, community.docker.* modules fail
    name: ___                                     # YOUR TASK: package name (it's not "docker-py" anymore)
    state: ___                                    # YOUR TASK
```

Пояснение: здесь собирается полная установка Docker — предварительные пакеты, GPG-ключ, APT-репозиторий, пакеты docker-ce, шаблон `daemon.json`, запуск сервиса, добавление пользователя в группу `docker` и установка Docker SDK для Python.

The *whys* you must answer:

Вопросы «почему», на которые ты должен ответить:

- **Why `{{ ansible_distribution_release }}`?** It resolves to `noble` on Ubuntu 24.04 and `jammy` on 22.04 — one role, both distros. (Comes from `gather_facts: true` — leave it on.)
  * **Зачем `{{ ansible_distribution_release }}`?** На Ubuntu 24.04 это `noble`, на 22.04 — `jammy`: одна роль подходит обоим дистрибутивам. (Значение приходит из `gather_facts: true` — не выключай его.)
- **Why the `notify:` on the template task, not the apt task?** A package install is a one-shot. A config-file change should restart Docker only when the file *actually changes*. That's the handler's purpose: a fresh first run restarts Docker once; the second run sees the file unchanged and skips the handler entirely. No `changed`, no restart, no downtime.
  * **Почему `notify:` стоит на задаче с шаблоном, а не на задаче apt?** Установка пакетов — разовое действие. Изменение конфиг-файла должно перезапускать Docker только когда файл *действительно меняется*. В этом и смысл handler: первый запуск один раз перезапускает Docker; второй видит, что файл не изменился, и пропускает handler полностью. Нет `changed`, нет перезапуска, нет простоя.
- **Why `append: true` on the user task?** Without it, Ansible *replaces* the user's group list — booting them out of `sudo`, `adm`, every group they were already in. This is the single most common Ansible footgun on shared boxes.
  * **Почему `append: true` в задаче с пользователем?** Без него Ansible *заменяет* список групп пользователя — и пользователь вылетает из `sudo`, `adm` и всех групп, в которых был. Это самая частая «вилка» Ansible на общих серверах.

### 2.3 — Provisioning playbook

2.3 — плейбук подготовки

`playbooks/provision.yml` — thin by design, all logic lives in the roles:

`playbooks/provision.yml` — намеренно тонкий: вся логика живёт в ролях:

```yaml
---
- name: Provision web servers
  hosts: ___                                      # YOUR TASK: target group
  become: ___                                     # YOUR TASK: true at play level, OR false + per-task become — argue your choice
  gather_facts: ___                               # YOUR TASK: needed for ansible_distribution_release
  roles:
    - ___                                         # YOUR TASK
    - ___                                         # YOUR TASK
```

### 2.4 — Run and verify

2.4 — Запуск и проверка

```bash
ansible-playbook playbooks/provision.yml          # first real run
```

First run is mostly yellow (illustrative; your `changed` count will differ):

Первый запуск в основном жёлтый (пример; твоё число `changed` будет другим):

```text
PLAY RECAP **********************************************************
lab-vm : ok=11  changed=8  unreachable=0  failed=0
```

### 2.5 — Proof of work

2.5 — Подтверждение работы

Paste into `docs/LAB05.md`:

Вставь в `docs/LAB05.md`:

- Both files' content (`common/tasks/main.yml`, `docker/tasks/main.yml`) — fenced as YAML
  * содержимое обоих файлов (`common/tasks/main.yml`, `docker/tasks/main.yml`) — в блоке YAML
- The **full PLAY RECAP** line from `ansible-playbook playbooks/provision.yml`
  * **полную строку PLAY RECAP** из `ansible-playbook playbooks/provision.yml`
- `ansible webservers -a "docker --version"` output proving Docker is on the box
  * вывод `ansible webservers -a "docker --version"`, доказывающий, что Docker стоит на сервере
- Your one-paragraph answer to each *why* in 2.1 and 2.2
  * твой ответ одним абзацем на каждый вопрос «почему» из 2.1 и 2.2

---

## Task 3 — Application Deployment Role + Vault (3 pts)

Задание 3 — роль развёртывания приложения и Vault (3 балла)

### 3.1 — Encrypt credentials with Ansible Vault

3.1 — Шифрование учётных данных с помощью Ansible Vault

```bash
ansible-vault create group_vars/all/vault.yml
```

You'll be prompted for a Vault password. **Save that password now** in a gitignored file so the 4 reruns in Task 4 don't make you retype it:

Ansible запросит пароль Vault. **Сохрани этот пароль сейчас** в gitignored-файле, чтобы не вводить его заново при четырёх перезапусках в задании 4:

```bash
echo 'your-vault-password' > .vault_pass
chmod 600 .vault_pass
# .vault_pass is already in your .gitignore (see "How to Submit")
```

Tell Ansible to use it (one of two ways — pick one):

Укажи Ansible, где брать пароль (один из двух способов — выбери один):

- Per-command: `ansible-playbook playbooks/deploy.yml --vault-password-file .vault_pass`
  * Для каждой команды: `ansible-playbook playbooks/deploy.yml --vault-password-file .vault_pass`
- In `ansible.cfg`: add `vault_password_file = .vault_pass` under `[defaults]`
  * В `ansible.cfg`: добавь `vault_password_file = .vault_pass` в секцию `[defaults]`

(The `--ask-vault-pass` form in §3.4 below is the prompt-each-time alternative — use whichever you prefer.)

(Форма с `--ask-vault-pass` в §3.4 ниже — вариант с запросом пароля каждый раз; используй то, что тебе удобнее.)

`YOUR TASK`: put your Docker Hub credentials and app config inside (the file is AES-256 encrypted at rest — safe to commit; the password is NOT).

`YOUR TASK`: положи туда учётные данные Docker Hub и конфигурацию приложения (файл шифруется AES-256 — его безопасно коммитить; а вот пароль — нет).

```yaml
---
dockerhub_username: ___                           # YOUR TASK: your Docker Hub username
dockerhub_password: ___                           # YOUR TASK: an ACCESS TOKEN (dckr_pat_…), not your login password
docker_image: ___                                 # YOUR TASK: e.g. "{{ dockerhub_username }}/devops-info-service"
docker_image_tag: ___                             # YOUR TASK: prefer an immutable tag (a SHA) over "latest"
app_port: ___                                     # YOUR TASK
app_container_name: ___                           # YOUR TASK
```

Confirm it decrypts:

Убедись, что файл расшифровывается:

```bash
ansible-vault view group_vars/all/vault.yml
```

> Why a Docker Hub *access token*, not your account password? An access token is scoped (read-only / read-write / admin), revocable per-token, and shows up in your DH audit log. A password leak compromises the whole account.
>
> Почему *access token* Docker Hub, а не пароль аккаунта? Access token имеет ограниченные права (read-only / read-write / admin), отзывается отдельно и виден в audit log DH. Утечка пароля компрометирует весь аккаунт.

### 3.2 — `app_deploy` role

3.2 — роль `app_deploy`

`roles/app_deploy/defaults/main.yml`:

```yaml
---
app_restart_policy: ___                           # YOUR TASK: unless-stopped or on-failure — why?
app_health_path: ___                              # YOUR TASK: /health (matches your Lab 1 service)
```

`roles/app_deploy/handlers/main.yml`:

```yaml
---
- name: ___                                       # YOUR TASK: handler name (`restart app` is fine)
  community.docker.docker_container:              # module given
    name: ___                                     # YOUR TASK: {{ app_container_name }}
    state: ___                                    # YOUR TASK
    restart: ___                                  # YOUR TASK
```

`roles/app_deploy/tasks/main.yml` — module names given (this is the lecture's `community.docker.*` choice); args are the skill:

`roles/app_deploy/tasks/main.yml` — имена модулей даны (это лекционный выбор `community.docker.*`); аргументы — это навык:

```yaml
---
- name: ___                                       # YOUR TASK: log in to Docker Hub
  community.docker.docker_login:                  # module given
    username: ___                                 # YOUR TASK
    password: ___                                 # YOUR TASK
  no_log: ___                                     # YOUR TASK: true — why?

- name: ___                                       # YOUR TASK: pull the image
  community.docker.docker_image:                  # module given
    name: ___                                     # YOUR TASK: {{ docker_image }}
    tag: ___                                      # YOUR TASK
    source: ___                                   # YOUR TASK: pull
    force_source: ___                             # YOUR TASK: false — defend your answer (idempotency)

- name: ___                                       # YOUR TASK: run the container
  community.docker.docker_container:              # module given
    name: ___                                     # YOUR TASK
    image: ___                                    # YOUR TASK: "{{ docker_image }}:{{ docker_image_tag }}"
    state: ___                                    # YOUR TASK
    restart_policy: ___                           # YOUR TASK
    ports:
      - ___                                       # YOUR TASK: "host:container" — e.g. "{{ app_port }}:5000"
    pull: ___                                     # YOUR TASK: never/always/missing — why never on rerun?

- name: ___                                       # YOUR TASK: wait for the port to accept connections
  ansible.builtin.wait_for:                       # module given
    host: ___                                     # YOUR TASK
    port: ___                                     # YOUR TASK
    timeout: ___                                  # YOUR TASK

- name: ___                                       # YOUR TASK: verify /health returns HTTP 200
  ansible.builtin.uri:                            # module given
    url: ___                                      # YOUR TASK: full URL ending in {{ app_health_path }}
    status_code: ___                              # YOUR TASK
    return_content: ___                           # YOUR TASK
  register: ___                                   # YOUR TASK: capture the response so docs can show it
```

Пояснение: роль логинится в Docker Hub, забирает образ, запускает контейнер, ждёт открытия порта и проверяет `/health`; обрати внимание на `no_log` и `pull`.

The *whys* (one sentence each in `docs/LAB05.md`):

Вопросы «почему» (по одному предложению на каждый в `docs/LAB05.md`):

- **`no_log: true` on the login task** — without it, your Docker Hub token shows up plaintext in the CI build log. CI logs are world-readable on public repos. There is no "delete" button.
  * **`no_log: true` на задаче логина** — без этого твой токен Docker Hub попадёт открытым текстом в лог CI-сборки. Логи CI в публичных репозиториях читает весь мир. Кнопки «удалить» не существует.
- **`pull: never` on the run task** — the previous pull step already fetched the right image. `pull: always` on `docker_container` re-pulls every run, which (a) breaks idempotency cosmetically and (b) wastes a registry round-trip.
  * **`pull: never` на задаче запуска** — предыдущий шаг уже скачал нужный образ. `pull: always` на `docker_container` перекачивает образ при каждом запуске, что (а) косметически ломает идемпотентность и (б) тратит лишний сетевой запрос к registry.
- **Why the `community.docker` collection and not `community.general`?** Docker modules were forked into a dedicated collection in 2020 for faster release cadence. `community.general.docker_*` doesn't exist anymore — bare-Stack-Overflow answers may still reference it; they're stale.
  * **Почему коллекция `community.docker`, а не `community.general`?** Docker-модули выделили в отдельную коллекцию в 2020 году ради более частых релизов. `community.general.docker_*` больше не существует — ответы со Stack Overflow со старыми именами устарели.

### 3.3 — Deployment playbook

3.3 — плейбук развёртывания

`playbooks/deploy.yml`:

```yaml
---
- name: Deploy application
  hosts: ___                                      # YOUR TASK
  become: ___                                     # YOUR TASK: why true? what does the docker_container module need root for?
  gather_facts: ___                               # YOUR TASK
  roles:
    - ___                                         # YOUR TASK
```

### 3.4 — Run and verify

3.4 — Запуск и проверка

```bash
ansible-playbook playbooks/deploy.yml --ask-vault-pass    # or --vault-password-file .vault_pass
curl -fsS http://<VM-IP>:<app_port>/health | jq .         # expect HTTP 200 + JSON
```

Пояснение: плейбук запускается с паролем Vault, а `curl` проверяет эндпоинт `/health` и прогоняет JSON через `jq`.

### 3.5 — Proof of work

3.5 — Подтверждение работы

Paste into `docs/LAB05.md`:

Вставь в `docs/LAB05.md`:

- `app_deploy/tasks/main.yml` content (fenced YAML)
  * содержимое `app_deploy/tasks/main.yml` (в блоке YAML)
- The PLAY RECAP from `ansible-playbook playbooks/deploy.yml` (run #1)
  * PLAY RECAP из `ansible-playbook playbooks/deploy.yml` (запуск №1)
- `ansible webservers -a "docker ps --format '{% raw %}{{.Names}}\t{{.Status}}{% endraw %}'"` output (your container running)
  * вывод `ansible webservers -a "docker ps --format '{% raw %}{{.Names}}\t{{.Status}}{% endraw %}'"` (твой контейнер работает)
- `curl -s http://<VM-IP>:<port>/health | jq .` against the VM (HTTP 200, the real JSON your Lab 1 service emits)
  * `curl -s http://<VM-IP>:<port>/health | jq .` с VM (HTTP 200 и настоящий JSON, который отдаёт сервис из лабы 1)
- Proof Vault file is encrypted: `head -1 group_vars/all/vault.yml` showing `$ANSIBLE_VAULT;1.1;AES256`
  * доказательство, что Vault-файл зашифрован: `head -1 group_vars/all/vault.yml` показывает `$ANSIBLE_VAULT;1.1;AES256`

---

## Task 4 — Idempotency Proof (1 pt)

Задание 4 — доказательство идемпотентности (1 балл)

This is the headline acceptance test. Configuration management's killer feature is that running the same playbook twice has the same effect as running it once.

Это главный приёмочный тест. Убийственная особенность управления конфигурацией в том, что запуск одного и того же плейбука дважды даёт тот же результат, что и один запуск.

```bash
# Vault password file in place (see Task 3.1) OR add --ask-vault-pass to each line
ansible-playbook playbooks/provision.yml          # run #1
ansible-playbook playbooks/provision.yml          # run #2
ansible-playbook playbooks/deploy.yml             # run #1 (already done in Task 3)
ansible-playbook playbooks/deploy.yml             # run #2
```

Пояснение: каждый плейбук запускается дважды; приёмочный результат — `changed=0` во втором запуске.

> 💡 Once `group_vars/all/vault.yml` exists, **every** playbook (including `provision.yml`) loads it on start and needs the Vault password — even though `provision.yml` doesn't *use* the secrets. Either set `vault_password_file` in `ansible.cfg` once, or append `--ask-vault-pass` to all four commands above.
>
> 💡 Как только `group_vars/all/vault.yml` существует, **каждый** плейбук (включая `provision.yml`) загружает его при старте и требует пароль Vault — хотя `provision.yml` и не *использует* секреты. Либо один раз задай `vault_password_file` в `ansible.cfg`, либо добавляй `--ask-vault-pass` ко всем четырём командам выше.

The **second run of each must report `changed=0`**. (Illustrative.)

**Второй запуск каждого плейбука должен показать `changed=0`**. (Пример.)

```text
# provision.yml, run #1
lab-vm : ok=11  changed=8  unreachable=0  failed=0
# provision.yml, run #2
lab-vm : ok=11  changed=0  unreachable=0  failed=0       # ← changed=0 = idempotent ✓

# deploy.yml, run #1
lab-vm : ok=6   changed=4  unreachable=0  failed=0
# deploy.yml, run #2
lab-vm : ok=6   changed=0  unreachable=0  failed=0       # ← changed=0 = idempotent ✓
```

If anything still changes on run #2, **the task is not idempotent — fix it**. Usual culprits:

Если при втором запуске что-то всё ещё меняется, **задача не идемпотентна — исправь её**. Обычные виновники:

- A stray `ansible.builtin.shell` or `command` (no state model). Wrap with `creates:`/`removes:`, or replace with the proper module.
  * лишний `ansible.builtin.shell` или `command` (нет модели состояния). Оберни его в `creates:`/`removes:` или замени подходящим модулем.
- A template that renders a timestamp or random value.
  * шаблон, который рендерит временную метку или случайное значение.
- `state: latest` on a package — the moment the upstream repo gets a new version, you'll see `changed=1` forever.
  * `state: latest` для пакета: как только в upstream-репозитории появится новая версия, ты навсегда увидишь `changed=1`.
- `force_source: true` on `docker_image` — always re-pulls.
  * `force_source: true` у `docker_image` — образ перекачивается всегда.

Confirm there's no drift with a dry-run:

Убедись, что дрейфа нет, с помощью dry-run:

```bash
ansible-playbook playbooks/provision.yml --check --diff
```

### 4.1 — Proof of work

4.1 — Подтверждение работы

Paste into `docs/LAB05.md`:

Вставь в `docs/LAB05.md`:

- Both PLAY RECAPs from `provision.yml` (run #1 changed=N, run #2 changed=0)
  * оба PLAY RECAP из `provision.yml` (запуск №1 changed=N, запуск №2 changed=0)
- Both PLAY RECAPs from `deploy.yml` (same shape)
  * оба PLAY RECAP из `deploy.yml` (в том же виде)
- Output of the `--check --diff` dry-run (no changes, no diffs)
  * вывод dry-run `--check --diff` (изменений нет, диффов нет)
- A one-paragraph analysis: which tasks changed on run #1, why they reported `ok` on run #2, and if anything stayed `changed=1` — what you fixed
  * анализ одним абзацем: какие задачи изменились при первом запуске, почему при втором они показали `ok` и что ты исправил, если что-то осталось `changed=1`

---

## Task 5 — Documentation (1 pt)

Задание 5 — документация (1 балл)

`ansible/docs/LAB05.md` — required sections, in order:

`ansible/docs/LAB05.md` — обязательные разделы по порядку:

1. **Architecture** — `ansible-core` version, target OS, why three roles (not one monolithic playbook)
   * **Архитектура** — версия `ansible-core`, целевая ОС, почему три роли (а не один монолитный плейбук)
2. **Roles** — for each of `common`, `docker`, `app_deploy`: purpose, key variables, handler behaviour
   * **Роли** — для каждой из `common`, `docker`, `app_deploy`: назначение, ключевые переменные, поведение handler
3. **The *why* answers** from Tasks 2 and 3 (FQCN, selective `become`, `state: present` vs `latest`, `append: true`, handler on template vs on apt, `no_log`, `pull: never`)
   * **Ответы на вопросы «почему»** из заданий 2 и 3 (FQCN, точечный `become`, `state: present` против `latest`, `append: true`, handler на шаблоне против apt, `no_log`, `pull: never`)
4. **Idempotency proof** — both PLAY RECAPs (Task 4) with one-paragraph analysis
   * **Доказательство идемпотентности** — оба PLAY RECAP (задание 4) с анализом в один абзац
5. **Vault** — how credentials are stored, password-handling strategy, the encrypted-header proof
   * **Vault** — как хранятся учётные данные, стратегия работы с паролем, доказательство зашифрованного заголовка
6. **Deployment verification** — `docker ps` and the `/health` `curl` output
   * **Проверка развёртывания** — `docker ps` и вывод `curl` для `/health`

---

## Bonus Task — Dynamic Inventory (2 pts)

Бонусное задание — динамический inventory (2 балла)

Discover your Lab 4 VM(s) automatically via a cloud inventory plugin instead of hardcoding the IP. A static list rots the moment a VM is recreated and gets a new address — the inventory has to know that *before* Ansible can configure anything.

Обнаруживай VM из лабы 4 автоматически через плагин облачного inventory вместо жёстко прописанного IP. Статический список устаревает в момент, когда VM пересоздают и она получает новый адрес, — а inventory должен знать об этом *до того*, как Ansible что-то начнёт настраивать.

`YOUR TASK`:

`YOUR TASK`:

1. Install the collection for your Lab 4 cloud:
   * Установи коллекцию для облака твоей лабы 4:
   ```bash
   ansible-galaxy collection install ___        # YOUR TASK: amazon.aws / google.cloud / azure.azcollection / yandex.cloud
   ```
2. Write `inventory/<cloud>.yml` that:
   * Напиши `inventory/<cloud>.yml`, который:
   - Sets `plugin:` to the FQCN inventory plugin (e.g. `amazon.aws.aws_ec2`)
     * Выставляет `plugin:` в FQCN плагина inventory (например, `amazon.aws.aws_ec2`)
   - Filters to **running** instances tagged for this course (use `filters:`)
     * фильтрует только **работающие** инстансы с тегом этого курса (используй `filters:`)
   - Uses `compose:` to set `ansible_host` to the public IP (the field name varies — see hints below)
     * использует `compose:` для `ansible_host` = публичный IP (имя поля зависит от облака — смотри подсказки ниже)
   - Uses `keyed_groups:` to build a group automatically (e.g. `webservers` from a `Role` tag)
     * использует `keyed_groups:` для автоматического создания группы (например, `webservers` из тега `Role`)
3. Point Ansible at it (`-i inventory/<cloud>.yml` or update `ansible.cfg`)
   * Укажи Ansible на этот файл (`-i inventory/<cloud>.yml` или обнови `ansible.cfg`)
4. Run:
   * Запусти:
   ```bash
   ansible-inventory -i inventory/<cloud>.yml --graph
   ansible all -i inventory/<cloud>.yml -m ansible.builtin.ping
   ansible-playbook -i inventory/<cloud>.yml playbooks/provision.yml --check --diff
   ```

Skeleton (AWS — adapt field names for your cloud):

Скелет (AWS — адаптируй имена полей под своё облако):

```yaml
# inventory/aws.yml
plugin: ___                                       # YOUR TASK: amazon.aws.aws_ec2
regions:
  - ___                                           # YOUR TASK
filters:
  ___: ___                                        # YOUR TASK: tag:Project=devops-core
  instance-state-name: ___                        # YOUR TASK: running
keyed_groups:
  - key: ___                                      # YOUR TASK: tags.Role
    prefix: ___                                   # YOUR TASK
compose:
  ansible_host: ___                               # YOUR TASK: public_ip_address (for AWS)
```

Пояснение: плагин `aws_ec2` строит inventory по тегам и состоянию инстансов, а `compose` сопоставляет `ansible_host` с публичным IP.

<details>
<summary>💡 Per-cloud field hints</summary>

💡 Подсказки по полям для каждого облака

| Cloud | Collection | Plugin | Public-IP field |
|---|---|---|---|
| AWS | `amazon.aws` | `aws_ec2` | `public_ip_address` |
| GCP | `google.cloud` | `gcp_compute` | `networkInterfaces[0].accessConfigs[0].natIP` |
| Azure | `azure.azcollection` | `azure_rm` | `public_ipv4_addresses[0]` |
| Yandex | `yandex.cloud` | `yandex_compute` | nested under `network_interfaces[0]` |

| Облако | Коллекция | Плагин | Поле публичного IP |
|---|---|---|---|
| AWS | `amazon.aws` | `aws_ec2` | `public_ip_address` |
| GCP | `google.cloud` | `gcp_compute` | `networkInterfaces[0].accessConfigs[0].natIP` |
| Azure | `azure.azcollection` | `azure_rm` | `public_ipv4_addresses[0]` |
| Yandex | `yandex.cloud` | `yandex_compute` | вложено в `network_interfaces[0]` |

Docs: [Inventory plugins](https://docs.ansible.com/ansible-core/2.21/plugins/inventory.html) · [aws_ec2](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)

Документация: [Inventory plugins](https://docs.ansible.com/ansible-core/2.21/plugins/inventory.html) · [aws_ec2](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)

</details>

### Bonus proof of work

Подтверждение работы по бонусу

Paste into `docs/LAB05.md`:

Вставь в `docs/LAB05.md`:

- The full `inventory/<cloud>.yml` (no secrets — auth is via env vars / IAM role)
  * полный `inventory/<cloud>.yml` (без секретов — аутентификация через переменные окружения / IAM-роль)
- `ansible-inventory --graph` output showing the auto-discovered host(s)
  * вывод `ansible-inventory --graph`, показывающий автоматически найденные хосты
- `ansible all -i inventory/<cloud>.yml -m ansible.builtin.ping` PLAY RECAP
  * PLAY RECAP из `ansible all -i inventory/<cloud>.yml -m ansible.builtin.ping`
- One paragraph: which plugin you used and why, how you authenticated (env vars? IAM instance profile? cloud SDK config?), and what happens when a VM's IP changes (nothing — no manual edits)
  * один абзац: какой плагин ты использовал и почему, как аутентифицировался (переменные окружения? IAM instance profile? конфиг cloud SDK?), и что происходит, когда IP VM меняется (ничего — ручных правок нет)

---

## How to Submit

Как сдать работу

```bash
git switch -c lab05

# Keep secrets out of Git
cat >> .gitignore <<'EOF'
# Ansible
*.retry
.vault_pass
__pycache__/
EOF

git add ansible/ .gitignore
git commit -m "feat(lab05): ansible roles — common, docker, app_deploy + idempotency proof"
git push -u origin lab05
```

Пояснение: создаём ветку, добавляем `.vault_pass` и другие секреты в `.gitignore`, затем коммитим только каталог `ansible/`.

**Verify before pushing:** `.vault_pass` absent, SSH private keys absent, only the *encrypted* `vault.yml` committed.

**Проверь перед пушем:** `.vault_pass` отсутствует, приватные SSH-ключи отсутствуют, закоммичен только *зашифрованный* `vault.yml`.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab05` → `course-repo:master` *(reviewed)*
  * `your-fork:lab05` → `course-repo:master` *(с ревью)*
- `your-fork:lab05` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab05` → `your-fork:master` *(сливается в твой основной репозиторий по завершении)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 done — inventory + ansible.cfg + ping succeeds
- [ ] Task 2 done — common + docker roles applied; PLAY RECAP captured
- [ ] Task 3 done — app_deploy role + encrypted Vault; /health curl is HTTP 200
- [ ] Task 4 done — idempotency: both run #2 PLAY RECAPs show changed=0
- [ ] Task 5 done — docs/LAB05.md covers all six sections with the *why* answers
- [ ] Bonus — dynamic inventory: ansible-inventory --graph + ping against it
```

---

## Acceptance Criteria

Критерии приёмки

### Task 1 (2 pts)

Задание 1 (2 балла)

- ✅ `ansible-core` 2.21 installed (`ansible --version`)
  * ✅ установлен `ansible-core` 2.21 (`ansible --version`)
- ✅ Three roles scaffolded (`common`, `docker`, `app_deploy`) with conventional subdirs
  * ✅ создан каркас трёх ролей (`common`, `docker`, `app_deploy`) с привычными подкаталогами
- ✅ Static inventory + `ansible.cfg` configured
  * ✅ настроены статический inventory и `ansible.cfg`
- ✅ `ansible all -m ansible.builtin.ping` returns `pong`
  * ✅ `ansible all -m ansible.builtin.ping` возвращает `pong`

### Task 2 (3 pts)

Задание 2 (3 балла)

- ✅ `common` role installs base packages, sets timezone — passes with `become: true` only where root is needed
  * ✅ роль `common` устанавливает базовые пакеты и часовой пояс — проходит с `become: true` только там, где нужен root
- ✅ `docker` role installs Docker Engine, renders `daemon.json`, manages the service, adds the user to the `docker` group with `append: true`
  * ✅ роль `docker` устанавливает Docker Engine, рендерит `daemon.json`, управляет сервисом, добавляет пользователя в группу `docker` с `append: true`
- ✅ All tasks use FQCN modules
  * ✅ все задачи используют модули FQCN
- ✅ Handler restarts Docker **only** on template change (`notify:` on the template task)
  * ✅ handler перезапускает Docker **только** при изменении шаблона (`notify:` на задаче с шаблоном)
- ✅ `provision.yml` applies both roles cleanly
  * ✅ `provision.yml` чисто применяет обе роли

### Task 3 (3 pts)

Задание 3 (3 балла)

- ✅ Credentials stored in an encrypted Vault file (`head -1` shows `$ANSIBLE_VAULT;1.1;AES256`)
  * ✅ учётные данные хранятся в зашифрованном Vault-файле (`head -1` показывает `$ANSIBLE_VAULT;1.1;AES256`)
- ✅ `app_deploy` role pulls the Lab 2 image and runs it as a container
  * ✅ роль `app_deploy` забирает образ из лабы 2 и запускает его как контейнер
- ✅ `no_log: true` on the login task
  * ✅ `no_log: true` на задаче логина
- ✅ `/health` returns HTTP 200; `curl` output captured
  * ✅ `/health` возвращает HTTP 200; вывод `curl` приложен

### Task 4 (1 pt)

Задание 4 (1 балл)

- ✅ Run #1 PLAY RECAP shows `changed > 0`
  * ✅ PLAY RECAP первого запуска показывает `changed > 0`
- ✅ Run #2 PLAY RECAP shows `changed=0` — for **both** `provision.yml` and `deploy.yml`
  * ✅ PLAY RECAP второго запуска показывает `changed=0` — для **обоих** плейбуков `provision.yml` и `deploy.yml`
- ✅ `--check --diff` shows no drift
  * ✅ `--check --diff` не показывает дрейфа

### Task 5 (1 pt)

Задание 5 (1 балл)

- ✅ `docs/LAB05.md` covers all six sections
  * ✅ `docs/LAB05.md` покрывает все шесть разделов

### Bonus Task (2 pts)

Бонусное задание (2 балла)

- ✅ Cloud inventory plugin configured for the Lab 4 provider
  * ✅ облачный плагин inventory настроен для провайдера лабы 4
- ✅ `ansible-inventory --graph` shows auto-discovered hosts
  * ✅ `ansible-inventory --graph` показывает автоматически найденные хосты
- ✅ A playbook runs successfully against the dynamic inventory
  * ✅ плейбук успешно выполняется против динамического inventory

---

## Rubric

Критерии оценки

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Inventory & scaffolding | **2** | FQCN ping, cfg, three-role layout |
| **Task 2** — `common` + `docker` roles | **3** | Idempotent tasks, handler on template, FQCN, `append: true`, *whys* documented |
| **Task 3** — `app_deploy` + Vault | **3** | Lab 2 image runs, `/health` 200, encrypted Vault, `no_log` |
| **Task 4** — Idempotency | **1** | Run #2 `changed=0` on both playbooks |
| **Task 5** — Documentation | **1** | All six sections complete |
| **Bonus** — Dynamic inventory | **2** | Cloud plugin auto-discovers hosts |
| **Total** | **12** | 10 main + 2 bonus |

| Задание | Баллы | Критерии |
|------|-------:|----------|
| **Задание 1** — Inventory и каркас | **2** | FQCN ping, cfg, раскладка из трёх ролей |
| **Задание 2** — роли `common` + `docker` | **3** | Идемпотентные задачи, handler на шаблоне, FQCN, `append: true`, задокументированные *why* |
| **Задание 3** — `app_deploy` + Vault | **3** | образ из лабы 2 работает, `/health` = 200, зашифрованный Vault, `no_log` |
| **Задание 4** — Идемпотентность | **1** | `changed=0` при запуске №2 для обоих плейбуков |
| **Задание 5** — Документация | **1** | все шесть разделов заполнены |
| **Бонус** — динамический inventory | **2** | облачный плагин автоматически находит хосты |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Critical requirements:**

**Критические требования:**

- MUST use role-based structure (not a monolithic playbook)
  * ОБЯЗАТЕЛЬНО использовать структуру на ролях (а не монолитный плейбук)
- MUST use FQCN modules everywhere
  * ОБЯЗАТЕЛЬНО использовать модули FQCN везде
- MUST demonstrate idempotency (run #2 changed=0 for both playbooks — this is the lab's headline)
  * ОБЯЗАТЕЛЬНО продемонстрировать идемпотентность (`changed=0` при запуске №2 для обоих плейбуков — это главный результат лабы)
- MUST use Ansible Vault; MUST NOT commit `.vault_pass` or plaintext secrets
  * ОБЯЗАТЕЛЬНО использовать Ansible Vault; НЕЛЬЗЯ коммитить `.vault_pass` или секреты открытым текстом

---

## Resources

Ресурсы

<details>
<summary>📚 Ansible Core 2.21</summary>

- [ansible-core 2.21 docs](https://docs.ansible.com/ansible-core/2.21/)
  * [Документация ansible-core 2.21](https://docs.ansible.com/ansible-core/2.21/)
- [Roles](https://docs.ansible.com/ansible-core/2.21/playbook_guide/playbooks_reuse_roles.html)
  * [Роли](https://docs.ansible.com/ansible-core/2.21/playbook_guide/playbooks_reuse_roles.html)
- [Handlers](https://docs.ansible.com/ansible-core/2.21/playbook_guide/playbooks_handlers.html)
  * [Handlers](https://docs.ansible.com/ansible-core/2.21/playbook_guide/playbooks_handlers.html)
- [Inventory](https://docs.ansible.com/ansible-core/2.21/inventory_guide/intro_inventory.html)
  * [Inventory](https://docs.ansible.com/ansible-core/2.21/inventory_guide/intro_inventory.html)
- [FQCN](https://docs.ansible.com/ansible-core/2.21/reference_appendices/glossary.html#term-FQCN)
  * [FQCN](https://docs.ansible.com/ansible-core/2.21/reference_appendices/glossary.html#term-FQCN)
- *Ansible for DevOps* — Jeff Geerling (2024 ed.)
  * *Ansible for DevOps* — Джефф Гирлинг (изд. 2024)

</details>

<details>
<summary>🔒 Vault & secrets</summary>

🔒 Vault и секреты

- [Ansible Vault](https://docs.ansible.com/ansible-core/2.21/vault_guide/index.html)
  * [Ansible Vault](https://docs.ansible.com/ansible-core/2.21/vault_guide/index.html)
- [no_log](https://docs.ansible.com/ansible-core/2.21/reference_appendices/logging.html)
  * [no_log](https://docs.ansible.com/ansible-core/2.21/reference_appendices/logging.html)

</details>

<details>
<summary>🐳 Docker modules & dynamic inventory</summary>

🐳 Docker-модули и динамический inventory

- [community.docker collection](https://docs.ansible.com/ansible/latest/collections/community/docker/index.html)
  * [Коллекция community.docker](https://docs.ansible.com/ansible/latest/collections/community/docker/index.html)
- [docker_login](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_login_module.html) · [docker_image](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_image_module.html) · [docker_container](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_container_module.html)
- [Install Docker on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
  * [Установка Docker в Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Inventory plugins](https://docs.ansible.com/ansible-core/2.21/plugins/inventory.html)
  * [Плагины inventory](https://docs.ansible.com/ansible-core/2.21/plugins/inventory.html)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных dry-run)

- **`become: true` without passwordless sudo** — the play fails with `sudo: a password is required`. Fix on the VM (cloud-init usually creates the login user with `NOPASSWD: ALL` in `/etc/sudoers.d/`); on hand-built hosts, add the user yourself. Bare-bones containers won't have it at all.
  * **`become: true` без безпарольного sudo** — play падает с `sudo: a password is required`. Исправь на VM (cloud-init обычно создаёт пользователя входа с `NOPASSWD: ALL` в `/etc/sudoers.d/`); на вручную собранных хостах добавь пользователя сам. В минимальных контейнерах этого может не быть вовсе.
- **`ansible.builtin.apt` on a non-Debian host** — fails immediately on RHEL/Fedora/Alpine. If your `common` role might ever target a non-Debian box, use `ansible.builtin.package` (auto-dispatches to apt/dnf/yum/zypper) and let the module pick. For Debian-specifics (PPA, repo files), keep `apt:` but gate with `when: ansible_os_family == "Debian"`.
  * **`ansible.builtin.apt` на не-Debian хосте** — сразу падает на RHEL/Fedora/Alpine. Если роль `common` когда-нибудь может нацелиться на не-Debian систему, используй `ansible.builtin.package` (сам выбирает apt/dnf/yum/zypper) и дай модулю решить. Для Debian-специфики (PPA, файлы репозиториев) оставляй `apt:`, но ограничивай через `when: ansible_os_family == "Debian"`.
- **`community.docker.docker_container` fails with `Failed to import the required Python library (Docker SDK for Python)`** — the *target* needs `pip install docker`. That's why the `docker` role's last task installs it via `ansible.builtin.pip`. Without it, `docker_container` errors before doing anything.
  * **`community.docker.docker_container` падает с `Failed to import the required Python library (Docker SDK for Python)`** — *целевой* машине нужен `pip install docker`. Поэтому последняя задача роли `docker` ставит его через `ansible.builtin.pip`. Без него `docker_container` выдаёт ошибку ещё до начала работы.
- **Dynamic inventory says `No inventory was parsed`** — the cloud collection isn't installed (`ansible-galaxy collection install amazon.aws`), the file isn't named `*.aws_ec2.yml` / `*.yml` matching `plugin:`, or your auth env vars (`AWS_ACCESS_KEY_ID` / `GOOGLE_APPLICATION_CREDENTIALS` / `AZURE_*`) aren't set. Run `ansible-inventory -i inventory/<cloud>.yml --list -vvv` to see the real error — `--graph` swallows it.
  * **Динамический inventory выдаёт `No inventory was parsed`** — коллекция облака не установлена (`ansible-galaxy collection install amazon.aws`), файл не назван `*.aws_ec2.yml` / `*.yml` в соответствии с `plugin:`, или не заданы переменные окружения для аутентификации (`AWS_ACCESS_KEY_ID` / `GOOGLE_APPLICATION_CREDENTIALS` / `AZURE_*`). Запусти `ansible-inventory -i inventory/<cloud>.yml --list -vvv`, чтобы увидеть настоящую ошибку — `--graph` её скрывает.
- **`append: true` forgotten on the user task** — Ansible *replaces* the user's group list, booting them out of `sudo`/`adm`. If you SSH'd in as `ubuntu` and ran that, the next login fails. Always `append: true` when adding to a group.
  * **Забыли `append: true` в задаче с пользователем** — Ansible *заменяет* список групп пользователя, и он вылетает из `sudo`/`adm`. Если ты зашёл по SSH как `ubuntu` и выполнил это, следующий вход не удастся. Всегда ставь `append: true` при добавлении в группу.
- **`state: latest` on a package** — every new upstream release flips run #2 from `changed=0` to `changed=1`. Use `state: present` for reproducible runs.
  * **`state: latest` для пакета** — каждый новый релиз upstream переключает запуск №2 с `changed=0` на `changed=1`. Используй `state: present` для воспроизводимых прогонов.
- **Vault file gets committed without `--encrypt`** — `git diff` shows your tokens in plaintext. Recovery: `git reset --hard HEAD~1`, rotate the token (it's burned), re-encrypt. Always `ansible-vault create` (not `vi`).
  * **Vault-файл закоммичен без `--encrypt`** — `git diff` показывает твои токены открытым текстом. Восстановление: `git reset --hard HEAD~1`, ротируй токен (он скомпрометирован), перешифруй. Всегда используй `ansible-vault create` (а не `vi`).
- **`provision.yml` suddenly prompts for a Vault password.** Once `group_vars/all/vault.yml` exists, Ansible loads it on every play — even `provision.yml` that doesn't reference the secrets. Either set `vault_password_file = .vault_pass` in `ansible.cfg` once, or pass `--ask-vault-pass`/`--vault-password-file` to *all four* runs in Task 4. Skipping this is the #1 reason students retype their password 8 times during the idempotency proof.
  * **`provision.yml` внезапно просит пароль Vault.** Как только `group_vars/all/vault.yml` существует, Ansible загружает его в каждом play — даже в `provision.yml`, который не ссылается на секреты. Либо один раз задай `vault_password_file = .vault_pass` в `ansible.cfg`, либо передавай `--ask-vault-pass`/`--vault-password-file` во *все четыре* запуска в задании 4. Пропуск этого — причина №1, по которой студенты восемь раз перенабирают пароль во время доказательства идемпотентности.

</details>

<details>
<summary>🛠️ Dev tools worth knowing</summary>

🛠️ Dev-инструменты, которые стоит знать

- [ansible-lint](https://ansible.readthedocs.io/projects/lint/) — catches FQCN violations, missing `name:`, `state: latest` smells. Lab 6's CI uses this.
  * [ansible-lint](https://ansible.readthedocs.io/projects/lint/) — ловит нарушения FQCN, отсутствующий `name:`, «запашки» `state: latest`. CI лабы 6 использует его.
- [Molecule](https://ansible.readthedocs.io/projects/molecule/) — spins each role in a container and tests it in isolation.
  * [Molecule](https://ansible.readthedocs.io/projects/molecule/) — поднимает каждую роль в контейнере и тестирует её изолированно.
- `ansible-playbook -vvv` — three v's print the SSH command + module JSON. Indispensable for debugging "why did it say `changed`?"
  * `ansible-playbook -vvv` — три «v» печатают SSH-команду и JSON модулей. Незаменимо для отладки вопроса «почему он сказал `changed`?»

</details>

---

## Looking Ahead

Что впереди

| Lab | What it adds to these roles |
|---:|---|
| 6 | Tags (`--tags docker`), blocks + `rescue:`/`always:`, Compose templates, CI workflow |
| 9 / 10 | Same image deployed to k3d via Helm — Ansible becomes the bootstrapper, not the deployer |
| 11 | Replace Ansible Vault with OpenBao as the runtime secret store |

| Лаба | Что добавляет к этим ролям |
|---:|---|
| 6 | Теги (`--tags docker`), блоки + `rescue:`/`always:`, шаблоны Compose, CI-воркфлоу |
| 9 / 10 | Тот же образ разворачивается в k3d через Helm — Ansible становится начальным загрузчиком, а не деплойером |
| 11 | Ansible Vault заменяется на OpenBao как runtime-хранилище секретов |

```mermaid
flowchart LR
  Lab4[🌍 Lab 4: VM] --> Lab5[🔧 Lab 5: roles + idempotency] --> Lab6[🚀 Lab 6: tags + Compose + CI] --> Lab9[☸️ Lab 9: K8s]
```

Схема: лаба 4 (VM) → лаба 5 (роли + идемпотентность) → лаба 6 (теги + Compose + CI) → лаба 9 (K8s).

---

**Good luck.** 🚀

**Удачи.** 🚀

> **Remember:** Tell the server *where to end up*, not *how* to get there. The acceptance test is `changed=0` on run #2. Keep credentials in Vault, keep roles small, document the *why*.
>
> **Запомни:** говори серверу, *каким он должен стать*, а не *как туда добраться*. Приёмочный тест — `changed=0` при запуске №2. Держи учётные данные в Vault, делай роли маленькими, документируй *почему*.
