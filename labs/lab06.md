# Lab 6 — Advanced Ansible & Continuous Deployment

Лаба 6 — Продвинутый Ansible и непрерывное развёртывание

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Ansible%20%26%20CI%2FCD-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Ansible%202.21%20|%20Docker%20Compose%20v2%20|%20GitHub%20Actions-informational)

> **Goal:** Turn the Lab 5 playbooks into a real deployment pipeline — refactor with `block`/`rescue`/`always`, template a Compose stack with Jinja2, add a double-gated wipe, and push-button deploy from GitHub Actions.
> **Deliverable:** A PR from `lab06` with refactored `ansible/roles/{common,docker,web_app}`, `.github/workflows/ansible-deploy.yml`, and `ansible/docs/LAB06.md` showing real proof (list-tags, rescue fired, idempotent PLAY RECAP).
>
> **Цель:** превратить плейбуки из лабы 5 в настоящий пайплайн развёртывания — переписать их с `block`/`rescue`/`always`, собрать стек Compose через шаблон Jinja2, добавить двойную защиту для вайпа и деплой по кнопке из GitHub Actions.
> **Что сдать:** PR из ветки `lab06` с переработанными `ansible/roles/{common,docker,web_app}`, файлом `.github/workflows/ansible-deploy.yml` и `ansible/docs/LAB06.md` с реальными доказательствами (список тегов, сработавший rescue, идемпотентный PLAY RECAP).

---

## Overview

Обзор

In Lab 5 you built roles that **configure** a server. In Lab 6 you make those same roles **deliver** an application — repeatedly, safely, and from CI. You will:

В лабе 5 ты собрал роли, которые **настраивают** сервер. В лабе 6 те же роли начинают **доставлять** приложение — повторяемо, безопасно и из CI. Ты научишься:

- Refactor `common` and `docker` with `block`/`rescue`/`always` and a deliberate tag taxonomy
  * Переработать `common` и `docker` с `block`/`rescue`/`always` и продуманной таксономией тегов
- Replace `docker run` with a Jinja2-templated Compose stack via `community.docker.docker_compose_v2`
  * Заменить `docker run` на стек Compose, собранный по шаблону Jinja2, через `community.docker.docker_compose_v2`
- Add a clean-reinstall button (`web_app_wipe`) that is gated **twice** so a single typo can't destroy prod
  * Добавить кнопку чистой переустановки (`web_app_wipe`) с **двойной** защитой, чтобы одна опечатка не уничтожила прод
- Wire `.github/workflows/ansible-deploy.yml` to lint → deploy → verify on every push to `ansible/**`
  * Настроить `.github/workflows/ansible-deploy.yml` на цепочку lint → deploy → verify при каждом пуше в `ansible/**`

> ⚠️ **Scope:** the lab targets one VM (yours from Lab 4) plus localhost for offline practice. Multi-host rolling rollouts are discussed but proven only structurally — same patterns transfer.
>
> ⚠️ **Охват:** лаба рассчитана на одну VM (твою из лабы 4) плюс localhost для тренировки офлайн. Многохостовые rolling-выкатки обсуждаются, но доказываются только структурно — паттерны переносятся один в один.

**What stays from Lab 5:** YAML, role layout, inventory, group_vars, Vault. We do **not** re-teach them.

**Что остаётся из лабы 5:** YAML, структура ролей, inventory, group_vars, Vault. Мы **не** объясняем их заново.

---

## Project State

Состояние проекта

**You have from previous labs:**

**Что у тебя есть из предыдущих лаб:**

- The DevOps Info Service (Lab 1) on a registry (Lab 2/3)
  * Сервис DevOps Info Service (лаба 1) в реестре (лабы 2/3)
- A target VM with SSH access (Lab 4)
  * Целевая VM с доступом по SSH (лаба 4)
- Ansible roles `common`, `docker`, `app_deploy`, an inventory, group_vars + Vault (Lab 5)
  * Роли Ansible `common`, `docker`, `app_deploy`, inventory, group_vars + Vault (лаба 5)

**This lab adds:**

**Что добавляет эта лаба:**

- `ansible/roles/{common,docker}` refactored with blocks + tags
  * `ansible/roles/{common,docker}` переработаны с блоками и тегами
- `ansible/roles/web_app/` (renamed from `app_deploy`) with `meta/main.yml`, `templates/docker-compose.yml.j2`, `tasks/main.yml`, `tasks/wipe.yml`, `defaults/main.yml`
  * `ansible/roles/web_app/` (переименована из `app_deploy`) с `meta/main.yml`, `templates/docker-compose.yml.j2`, `tasks/main.yml`, `tasks/wipe.yml`, `defaults/main.yml`
- `.github/workflows/ansible-deploy.yml`
  * `.github/workflows/ansible-deploy.yml`
- `ansible/docs/LAB06.md` — your proof report
  * `ansible/docs/LAB06.md` — твой отчёт с доказательствами

---

## Setup

Настройка

```bash
ansible --version | head -1                  # ansible-core 2.21.x
ansible-galaxy collection install community.docker    # 4.x
ansible-lint --version
docker compose version                       # v2.x (the plugin, NOT docker-compose v1)
```

Пояснение: команды проверяют версии ansible-core, коллекции community.docker, ansible-lint и Docker Compose — лаба рассчитана на ansible-core 2.21.x, community.docker 4.x и плагин Compose v2.

Rename the Lab 5 role before you start — the new name lines up with the wipe variable in Task 3:

Переименуй роль из лабы 5 до начала работы — новое имя совпадает с переменной вайпа из задачи 3:

```bash
cd ansible/roles && git mv app_deploy web_app
# update every reference: playbook roles:, group_vars keys, docs
```

Пояснение: роль переименовывается до начала работы; после `git mv` обнови все ссылки на старое имя — секцию `roles:`, ключи group_vars и документацию.

> ⚠️ **More than a `mv`.** Lab 5's `app_deploy` used `community.docker.docker_container` to run a single container. Task 2 replaces that with a Compose deployment, so `tasks/main.yml` and `handlers/main.yml` get **rewritten**, not edited. Keep the Vault file + the role's `defaults/main.yml` skeleton — but expect to **rename variables** to match Lab 6's new keys (`app_port` → `app_host_port` / `app_internal_port`, `docker_image_tag` → `docker_tag`, `app_container_name` → `app_name`). Grep `group_vars/` and `inventory/` for the old names after you write `defaults/main.yml` in §2.4.
>
> ⚠️ **Это больше, чем `mv`.** В лабе 5 `app_deploy` использовала `community.docker.docker_container`, чтобы запустить один контейнер. Задача 2 заменяет это деплоем через Compose, поэтому `tasks/main.yml` и `handlers/main.yml` **переписываются** заново, а не правятся. Сохрани файл Vault и заготовку `defaults/main.yml` роли — но готовься **переименовать переменные** под новые ключи лабы 6 (`app_port` → `app_host_port` / `app_internal_port`, `docker_image_tag` → `docker_tag`, `app_container_name` → `app_name`). После того как напишешь `defaults/main.yml` в §2.4, поищи старые имена через grep по `group_vars/` и `inventory/`.

---

## Task 1 — Refactor with Blocks & Tags (2 pts)

Задача 1 — Переработка с блоками и тегами (2 балла)

Lab 5's `common` and `docker` were linear task lists. You will restructure them so that (a) a transient apt/GPG flake doesn't kill the run, (b) you can target a single phase from the CLI, and (c) the `systemd` enable step runs **even when the install fails**.

В лабе 5 `common` и `docker` были линейными списками задач. Ты перестроишь их так, чтобы: (а) случайный сбой apt/GPG не убивал весь запуск, (б) можно было запустить одну фазу из CLI, и (в) шаг включения `systemd` выполнялся **даже при неудачной установке**.

### 1.1 Tag strategy (three axes)

1.1 Стратегия тегов (три оси)

Adopt three orthogonal tag axes and apply them consistently across both roles. This is the single decision that decides whether `--tags` is useful or noise.

Прими три ортогональные оси тегов и применяй их единообразно в обеих ролях. Именно это решение определяет, будет ли `--tags` полезен или превратится в шум.

| Axis | Tag names you must use | Used for |
|------|------------------------|----------|
| Component | `common`, `docker`, `web_app` | "Touch one role only" |
| Action | `install`, `config`, `deploy`, `wipe` | "Run one phase across roles" |
| Risk gate | `web_app_wipe` | Dangerous, opt-in only (Task 3) |

| Ось | Имена тегов, которые нужно использовать | Для чего |
|-----|------------------------------------------|----------|
| Компонент | `common`, `docker`, `web_app` | «Затронуть только одну роль» |
| Действие | `install`, `config`, `deploy`, `wipe` | «Выполнить одну фазу во всех ролях» |
| Защита от риска | `web_app_wipe` | Опасно, только по явному запросу (задача 3) |

`YOUR TASK`: in `ansible/docs/LAB06.md`, write a 3-line table showing which two-or-three tags each block gets. Example row: *"`roles/docker` install block → `[docker, docker_install, install]`."* This forces you to commit to the taxonomy **before** sprinkling tags.

**Твоя задача:** в `ansible/docs/LAB06.md` напиши таблицу из трёх строк, показывающую, какие два-три тега получает каждый блок. Пример строки: *«блок install в `roles/docker` → `[docker, docker_install, install]`»*. Так ты обяжешь себя определить таксономию **до** того, как начнёшь раскидывать теги.

### 1.2 Refactor the `docker` role

1.2 Переработка роли `docker`

**File:** `roles/docker/tasks/main.yml`

**Файл:** `roles/docker/tasks/main.yml`

Group the apt-key + repo + `docker-ce` install in a block. Give it a `rescue` that handles a flaky mirror/GPG fetch, and an `always` that guarantees the service is enabled — cleanup is sacred, it must run whether the install succeeded or not.

Собери установку apt-ключа, репозитория и пакета `docker-ce` в блок. Дай ему `rescue`, который переживает нестабильное зеркало или сбой загрузки GPG-ключа, и `always`, который гарантирует включение сервиса — очистка священна, она должна выполниться независимо от успеха установки.

`YOUR TASK`: fill in the shape below. Hints in the comments — do not paste solutions from Lab 5 unchanged; you are restructuring them.

**Твоя задача:** заполни заготовку ниже. Подсказки — в комментариях; не вставляй решения из лабы 5 без изменений, ты их перестраиваешь.

```yaml
- name: ___                                    # YOUR TASK: name the parent block
  block:
    - name: ___                                # YOUR TASK: add Docker apt GPG key
      ___:                                     # YOUR TASK: pick the right ansible.builtin module
        ___                                    # YOUR TASK: its args

    - name: ___                                # YOUR TASK: add Docker apt repo
      ___:
        ___

    - name: Install docker-ce
      ansible.builtin.apt:
        name: docker-ce
        update_cache: true
  rescue:
    - name: ___                                # YOUR TASK: what's a sensible recovery
                                               #            for a flaky mirror/GPG fetch?
      ___:
        ___
      retries: ___                             # YOUR TASK: pick a small int, justify in docs
      delay: ___
  always:
    - name: ___                                # YOUR TASK: ensure dockerd enabled + started
      ___:
        ___
  become: true
  tags: [___, ___, ___]                        # YOUR TASK: pick three tags from the taxonomy
```

Пояснение: это каркас блока `block`/`rescue`/`always` — заполни имена задач, модули и аргументы, отмеченные `YOUR TASK`; `rescue` должен переживать сбой зеркала/GPG, а `always` — включать и запускать `dockerd`.

> A tag on a `block` inherits to every task inside — do **not** re-tag each task. Lift `become: true` to the block, do not repeat it. If the `rescue` also fails, the host is marked failed and `always` still runs.
>
> Тег на `block` наследуется всеми задачами внутри — **не** помечай каждую задачу отдельно. Подними `become: true` на блок и не повторяй его. Если `rescue` тоже падает, хост помечается как failed, но `always` всё равно выполняется.

### 1.3 Refactor the `common` role

1.3 Переработка роли `common`

**File:** `roles/common/tasks/main.yml`

**Файл:** `roles/common/tasks/main.yml`

Apply the same shape with these requirements:

Примени ту же форму со следующими требованиями:

- The block installs your Lab 5 baseline packages (curl, gnupg, ca-certificates, …).
  * Блок устанавливает базовые пакеты из лабы 5 (curl, gnupg, ca-certificates, …).
- `rescue` runs `apt-get update --fix-missing` then retries.
  * `rescue` выполняет `apt-get update --fix-missing`, а затем повторяет попытку.
- `always` writes a small completion marker to `/tmp/lab6-common-ran` so you can prove the block executed end-to-end.
  * `always` пишет маленький маркер завершения в `/tmp/lab6-common-ran`, чтобы ты мог доказать, что блок отработал от начала до конца.
- Tags follow the taxonomy. Lift `become: true` to the block.
  * Теги соответствуют таксономии. Подними `become: true` на блок.

`YOUR TASK`: write the file. Same shape as 1.2, different module choices (`apt` package list, `command`/`shell` for the marker — pick the right one and justify).

**Твоя задача:** напиши файл. Форма та же, что в 1.2, но другие модули (`apt` для списка пакетов, `command`/`shell` для маркера — выбери правильный и обоснуй).

### 1.4 Prove a `rescue` actually fires

1.4 Докажи, что `rescue` действительно срабатывает

It's easy to write `rescue:` and never exercise it — and an unproven rescue path is worse than no rescue at all (you'll find out it doesn't work the day you need it).

Легко написать `rescue:` и ни разу его не проверить — а непроверенный rescue хуже, чем его отсутствие (ты узнаешь, что он не работает, именно в тот день, когда он понадобится).

`YOUR TASK`: deliberately break a task inside one of your blocks **once**, confirm the rescue runs, then revert. Capture the **rescue task's output** in the proof section — the recap line `rescued=1` alone is not enough.

**Твоя задача:** намеренно сломай одну задачу внутри одного из блоков **один раз**, убедись, что rescue сработал, и откати изменение. Сохрани **вывод задачи rescue** в разделе доказательств — одной строки `rescued=1` в recap недостаточно.

Two safe ways to force a failure (pick one — both fail the *block* task, both let `rescue` catch it, both revert cleanly):

Два безопасных способа вызвать сбой (выбери один — оба валят задачу *блока*, оба позволяют `rescue` перехватить ошибку и оба легко откатываются):

1. **Edit the role for one run.** Point `apt_repository:` at `http://does-not-exist.example.com/` (or set an apt key URL to a 404), run, then `git checkout roles/<role>/tasks/main.yml` to revert. Pro: most realistic — it's the exact failure mode rescue exists for.
   * **Измени роль на один запуск.** Направь `apt_repository:` на `http://does-not-exist.example.com/` (или подмени URL apt-ключа на 404), запусти, затем откати через `git checkout roles/<role>/tasks/main.yml`. Плюс: максимально реалистично — это ровно тот режим отказа, ради которого существует rescue.
2. **Use `--extra-vars` + a dummy assertion.** Add `ansible.builtin.fail: msg="forced"` gated by `when: simulate_failure | default(false) | bool` inside the block; run once with `-e simulate_failure=true`, capture proof, run again without it. Pro: no `git checkout` dance; the trigger lives in the role and you can re-run the proof on demand.
   * **Используй `--extra-vars` и фиктивную проверку.** Добавь в блок `ansible.builtin.fail: msg="forced"` с условием `when: simulate_failure | default(false) | bool`; запусти один раз с `-e simulate_failure=true`, сохрани доказательство, затем запусти без него. Плюс: не нужно плясать с `git checkout`; триггер живёт в роли, и доказательство можно воспроизвести в любой момент.

> **Do not** comment out a task or run with `--check` — neither exercises the rescue path. The block's task must actually FAIL (`failed=1` on a task line before the rescue line).
>
> **Не** закомментируй задачу и не запускай с `--check` — ни то, ни другое не проверяет путь rescue. Задача в блоке должна реально упасть (`failed=1` в строке задачи до строки rescue).

### 1.5 Verify selective execution

1.5 Проверь выборочное выполнение

```bash
ansible-playbook playbooks/provision.yml --list-tags                # shows the union of all tags
ansible-playbook playbooks/provision.yml --list-tasks --tags docker # tasks the next --tags docker run would run
ansible-playbook playbooks/provision.yml --tags install             # action axis: install-only across roles
ansible-playbook playbooks/provision.yml --skip-tags common         # component axis: everything but common
ansible-playbook playbooks/provision.yml --tags docker --check      # dry-run, component axis
```

Пояснение: первые две команды только показывают теги и задачи без запуска; остальные проверяют фильтрацию по осям «действие» и «компонент», а последняя — dry-run.

> `--list-tags` only prints the union — it does **not** prove your taxonomy is three-axis (component / action / risk-gate). Pair it with `--list-tasks --tags <axis>` from each axis to show selective filtering actually reaches the right slice. The taxonomy table from §1.1 carries the explanation.
>
> `--list-tags` печатает только объединение тегов — это **не** доказывает, что таксономия трёхосевая (компонент / действие / риск-гейт). Дополни его командой `--list-tasks --tags <axis>` по каждой оси, чтобы показать, что фильтрация действительно добирается до нужного среза. Объяснение несёт таблица таксономии из §1.1.

### 1.6 Proof of work — paste into `docs/LAB06.md`

1.6 Доказательства работы — вставь в `docs/LAB06.md`

- The **`--list-tags` output**, unedited. Tags from all three axes (component / action / risk-gate) must be present in the union.
  * **Вывод `--list-tags`** без правок. В объединении должны присутствовать теги всех трёх осей (компонент / действие / риск-гейт).
- One **selective `--tags`** run from EACH axis (e.g. `--tags docker`, `--tags install`, plus a `--tags web_app_wipe` dry-run) showing **most tasks skipped** in each — not "all 12 ran with tag=docker_install".
  * По одному запуску с **выборочным `--tags`** с КАЖДОЙ оси (например, `--tags docker`, `--tags install` и dry-run с `--tags web_app_wipe`), где видно, что **большинство задач пропущено**, — а не «все 12 выполнились с тегом docker_install».
- The **rescue path output** from 1.4 — the actual task name + result, not just `PLAY RECAP`.
  * **Вывод пути rescue** из 1.4 — реальное имя задачи и результат, а не только `PLAY RECAP`.
- Your taxonomy table from 1.1.
  * Твоя таблица таксономии из 1.1.

**Research (answer in your docs):**

**Исследование (ответь в своей документации):**

- How do tags inherit from a `block` into its tasks? Do tags on the role import combine or override?
  * Как теги наследуются от `block` к его задачам? Теги на импорте роли объединяются или переопределяются?
- Difference between the special `never` tag and a normal opt-in tag like `web_app_wipe`?
  * Чем специальный тег `never` отличается от обычного opt-in-тега вроде `web_app_wipe`?
- What happens to a host's downstream tasks when its `rescue` also fails?
  * Что происходит с последующими задачами хоста, если его `rescue` тоже падает?

---

## Task 2 — Upgrade to Docker Compose (3 pts)

Задача 2 — Переход на Docker Compose (3 балла)

Replace the Lab 5 `docker run` with a Jinja2-templated Compose stack, applied via `community.docker.docker_compose_v2`. The win is declarative state + free idempotency: re-running with the same rendered file shows `changed=0`.

Замени `docker run` из лабы 5 на стек Compose, отрендеренный из шаблона Jinja2 и применённый через `community.docker.docker_compose_v2`. Выигрыш — декларативное состояние и идемпотентность бесплатно: повторный запуск с тем же отрендеренным файлом даёт `changed=0`.

### 2.1 Declare the role dependency

2.1 Объяви зависимость роли

**File:** `roles/web_app/meta/main.yml`

**Файл:** `roles/web_app/meta/main.yml`

A `meta/main.yml` dependency makes `web_app` pull in `docker` automatically, in the right order, exactly once per play. You do not need a second `roles:` entry in your playbook.

Зависимость в `meta/main.yml` заставляет `web_app` автоматически подтягивать `docker` в правильном порядке и ровно один раз за play. Второй записи в `roles:` плейбука не нужно.

`YOUR TASK`: write this file. The shape:

**Твоя задача:** напиши этот файл. Форма:

```yaml
---
dependencies:
  - role: ___                                  # YOUR TASK: which role must run first?
    vars:
      ___: ["{{ ansible_user }}"]              # YOUR TASK: variable name from your
                                               #            Lab 5 docker role
```

Пояснение: `dependencies` перечисляет роли, которые выполняются до `web_app`; блок `vars` передаёт переменную из твоей роли `docker`.

### 2.2 Template the Compose file

2.2 Шаблон файла Compose

**File:** `roles/web_app/templates/docker-compose.yml.j2`

**Файл:** `roles/web_app/templates/docker-compose.yml.j2`

**Target render:** a working `docker-compose.yml` for **one** service that:

**Целевой результат:** рабочий `docker-compose.yml` для **одного** сервиса, который:

- pulls `{{ docker_image }}:{{ docker_tag }}`,
  * подтягивает `{{ docker_image }}:{{ docker_tag }}`,
- publishes `{{ app_host_port }}:{{ app_internal_port }}`,
  * публикует `{{ app_host_port }}:{{ app_internal_port }}`,
- injects every key/value from `{{ app_env }}` (a dict from group_vars) as environment variables,
  * передаёт каждую пару ключ/значение из `{{ app_env }}` (словарь из group_vars) как переменные окружения,
- restarts per `{{ restart_policy }}` (default `unless-stopped` — use a Jinja filter),
  * перезапускается согласно `{{ restart_policy }}` (по умолчанию `unless-stopped` — используй фильтр Jinja),
- ships a **healthcheck** hitting `/health` on `{{ app_internal_port }}` so `docker compose ps` shows `(healthy)`.
  * содержит **healthcheck**, обращающийся к `/health` на `{{ app_internal_port }}`, чтобы `docker compose ps` показывал `(healthy)`.

You must use **all three** Jinja2 mechanisms:

Ты должен использовать **все три** механизма Jinja2:

1. `{{ var }}` interpolation
   * Интерполяция `{{ var }}`
2. At least one `{% if %}` block (e.g. skip `environment:` when `app_env` is empty)
   * Как минимум один блок `{% if %}` (например, не выводить `environment:`, если `app_env` пуст)
3. At least one filter chain (e.g. `| default('...')` or `| to_nice_yaml`)
   * Как минимум одна цепочка фильтров (например, `| default('...')` или `| to_nice_yaml`)

`YOUR TASK`: write the template. The shape (intentionally incomplete — fill the body):

**Твоя задача:** напиши шаблон. Форма (намеренно неполная — заполни тело):

```jinja
{# Compose v2 — NO top-level `version:` field (dropped in 2023) #}
services:
  {{ ___ }}:                                   # YOUR TASK: service name var
    image: ___                                 # YOUR TASK: image:tag, two vars
    container_name: ___
    ports:
      - "___:___"                              # YOUR TASK: host:container, two vars
{% if ___ %}                                   # YOUR TASK: only render env block when app_env non-empty
    environment:
{% for ___, ___ in ___ %}                      # YOUR TASK: iterate the dict
      ___: "___"
{% endfor %}
{% endif %}
    restart: {{ ___ | default('___') }}        # YOUR TASK: var + the default value
    healthcheck:
      test: ___                                # YOUR TASK: curl/wget the in-container /health
      interval: ___                            # YOUR TASK: sensible default
      timeout: ___
      retries: ___
```

Пояснение: шаблон должен использовать интерполяцию, блок `{% if %}` для пустого `app_env` и фильтр `default` для политики перезапуска; поля `YOUR TASK` нужно заполнить.

> Compose v2 dropped the top-level `version:` key — if a tutorial still writes `version: '3.8'`, it predates 2023. Leave it off. Test the render locally with `ansible all -i localhost, -m template -a "src=... dest=/tmp/out.yml"` before running the full play.
>
> В Compose v2 убрали ключ `version:` верхнего уровня — если в уроке всё ещё пишут `version: '3.8'`, он старше 2023 года. Не добавляй его. Проверь рендер локально командой `ansible all -i localhost, -m template -a "src=... dest=/tmp/out.yml"` до запуска всего play.

### 2.3 Implement the deploy tasks

2.3 Реализуй задачи деплоя

**File:** `roles/web_app/tasks/main.yml`

**Файл:** `roles/web_app/tasks/main.yml`

The shape — fill the bodies:

Форма — заполни тела:

```yaml
---
- name: Include wipe tasks (gated — Task 3)
  ansible.builtin.include_tasks: wipe.yml
  tags: [web_app, web_app_wipe]

- name: ___                                    # YOUR TASK: name the deploy block
  block:
    - name: Create project directory
      ansible.builtin.file:
        path: "{{ compose_project_dir }}"
        state: directory
        mode: "0755"
      # NOTE: an aborted previous run can leave /tmp/<project> as a FILE — see Common Pitfalls

    - name: Render compose file
      ansible.builtin.template:
        src: ___                               # YOUR TASK: which template?
        dest: "___/docker-compose.yml"         # YOUR TASK: where on the target?
        mode: "0644"

    - name: ___                                # YOUR TASK: name this — "docker compose up -d"
      ___:                                     # YOUR TASK: the FQCN module — NOT the v1 module
                                               #            (Common Pitfalls: v1 vs v2)
        project_src: "{{ compose_project_dir }}"
        state: ___                             # YOUR TASK: which state means "up"?
        pull: ___
        remove_orphans: ___

    - name: Smoke test /health
      ansible.builtin.uri:
        url: "___"                             # YOUR TASK: build the URL from app_host_port
        status_code: 200
      register: hc
      retries: ___                             # YOUR TASK: bounded, not infinite
      delay: ___
      until: ___                               # YOUR TASK: condition on hc
  rescue:
    - name: ___                                # YOUR TASK: surface the failure
                                               #            (debug, fail, slack — your call)
      ___:
  tags: [___, ___, ___]                        # YOUR TASK: three tags from the taxonomy
```

Пояснение: `include_tasks` подключает wipe-задачи перед деплоем; блок создаёт каталог проекта, рендерит compose-файл, поднимает стек через `docker_compose_v2` и делает smoke-тест `/health`, а `rescue` сообщает об ошибке.

Setup notes (these don't need YOUR-TASK; they are plumbing):

Заметки по настройке (здесь нет YOUR TASK, это служебная часть):

- Collection installed on the **controller**: `ansible-galaxy collection install community.docker`.
  * Коллекция установлена на **контроллере**: `ansible-galaxy collection install community.docker`.
- `python3-docker` on the **target** (your `docker` role does this).
  * `python3-docker` на **целевой машине** (это делает твоя роль `docker`).
- Re-running the same play with no changes must show `changed=0`. If it doesn't, something in your template depends on a mutable input (timestamp, random, unsorted dict iteration).
  * Повторный запуск того же play без изменений должен показать `changed=0`. Если нет — что-то в шаблоне зависит от изменчивого входа (время, случайность, неупорядоченный обход словаря).

### 2.4 Variables

2.4 Переменные

**File:** `roles/web_app/defaults/main.yml`

**Файл:** `roles/web_app/defaults/main.yml`

`YOUR TASK`: declare the defaults. The keys you need: `app_name`, `docker_image`, `docker_tag`, `app_host_port`, `app_internal_port`, `compose_project_dir`, `restart_policy`, `app_env`. Use sensible defaults. **Do not** default `docker_tag` to `latest` for prod — pin it per-env in `group_vars`. Keep real secrets in your Lab 5 Vault file (`group_vars/.../vault.yml`) and merge them into `app_env` from there.

**Твоя задача:** объяви значения по умолчанию. Нужные ключи: `app_name`, `docker_image`, `docker_tag`, `app_host_port`, `app_internal_port`, `compose_project_dir`, `restart_policy`, `app_env`. Задай разумные значения. **Не** ставь `docker_tag` по умолчанию в `latest` для прода — фиксируй его для каждого окружения в `group_vars`. Настоящие секреты храни в файле Vault из лабы 5 (`group_vars/.../vault.yml`) и подмешивай их оттуда в `app_env`.

> ⚠️ **Migrate the Lab 5 names** at the same time. Lab 5 used `app_port`, `app_container_name`, `docker_image_tag` — those collide or just plain confuse against Lab 6's `app_host_port` / `app_internal_port`, `app_name`, `docker_tag`. After you write `defaults/main.yml`, `grep -RIn 'app_port\|app_container_name\|docker_image_tag' ansible/` and rename anything still using the old keys (group_vars, host_vars, vault). Leaving both around silently breaks templating.
>
> ⚠️ **Перенеси имена из лабы 5** сразу же. В лабе 5 использовались `app_port`, `app_container_name`, `docker_image_tag` — они конфликтуют или просто путаются с `app_host_port` / `app_internal_port`, `app_name`, `docker_tag` из лабы 6. После написания `defaults/main.yml` выполни `grep -RIn 'app_port\|app_container_name\|docker_image_tag' ansible/` и переименуй всё, что ещё использует старые ключи (group_vars, host_vars, vault). Если оставить оба набора, шаблонизация молча ломается.

### 2.5 Verify

2.5 Проверка

```bash
ansible-playbook playbooks/deploy.yml --tags deploy            # run #1: changed
ansible-playbook playbooks/deploy.yml --tags deploy            # run #2: ok (changed=0)
# On the target VM. Substitute YOUR values from group_vars/all.yml (not Jinja —
# Jinja is only rendered inside Ansible-managed files, not in your shell).
docker compose -f "<compose_project_dir>/docker-compose.yml" ps   # (healthy)
curl -fsS "http://<VM-IP>:<app_host_port>/health" | jq .
# Or run the same checks via Ansible, where {{...}} IS rendered:
# ansible all -m shell -a 'docker compose -f {{ compose_project_dir }}/docker-compose.yml ps'
```

Пояснение: первый запуск должен показать изменения, второй — `changed=0`; `docker compose ps` и `curl` выполняются на целевой VM, а не на контроллере, поэтому `{{...}}` в shell не раскрываются.

### 2.6 Proof of work — paste into `docs/LAB06.md`

2.6 Доказательства работы — вставь в `docs/LAB06.md`

- The **rendered** `docker-compose.yml` (post-template) — at least the first 20 lines.
  * **Отрендеренный** `docker-compose.yml` (после шаблона) — хотя бы первые 20 строк.
- Both PLAY RECAPs side-by-side — run #1 `changed≥1`, run #2 `changed=0`. **This is the idempotency proof.** A second run that still shows `changed=2` means your template renders differently each time (sort your dict keys, drop timestamps).
  * Оба PLAY RECAP рядом — запуск №1 `changed≥1`, запуск №2 `changed=0`. **Это и есть доказательство идемпотентности.** Если второй запуск всё ещё показывает `changed=2`, шаблон рендерится каждый раз по-разному (отсортируй ключи словаря, убери временные метки).
- `docker compose ps` showing `(healthy)`.
  * Вывод `docker compose ps` со статусом `(healthy)`.
- `curl /health` output with your real hostname/uptime.
  * Вывод `curl /health` с твоим реальным hostname/uptime.

**Research:**

**Исследование:**

- `restart: always` vs `unless-stopped` — when does the difference actually bite?
  * `restart: always` против `unless-stopped` — когда разница реально проявляется?
- Why does `python3-docker` go on the **target**, not the controller?
  * Почему `python3-docker` ставится на **целевую машину**, а не на контроллер?
- Why is one templated Compose file better than N `docker run` tasks?
  * Почему один шаблонный файл Compose лучше, чем N задач `docker run`?

---

## Task 3 — Double-Gated Wipe Logic (2 pts)

Задача 3 — Логика вайпа с двойной защитой (2 балла)

A clean reinstall is sometimes the only safe rollback: corrupted volume, rotated secret, broken cache. You need the button, **and** you need it gated so a sleepy 2am `--tags` typo can't destroy prod.

Иногда чистая переустановка — единственный безопасный откат: повреждённый том, ротированный секрет, битый кэш. Тебе нужна кнопка — **и** защита на ней, чтобы сонная опечатка в `--tags` в два часа ночи не уничтожила прод.

### 3.1 The double gate

3.1 Двойной гейт

Two **independent** gates: a `when:` variable check **and** an opt-in tag. Both must align for the wipe to run.

Два **независимых** гейта: проверка переменной через `when:` **и** opt-in-тег. Вайп выполняется, только если совпали оба.

| Invocation | Wipe? | Why |
|------------|-------|-----|
| `deploy.yml` | No | var false, tag not requested |
| `deploy.yml --tags web_app_wipe` | No | `when` blocks it (var still false) |
| `deploy.yml -e web_app_wipe=true` | Yes — wipe **then** deploy | clean reinstall |
| `deploy.yml -e web_app_wipe=true --tags web_app_wipe` | Yes — wipe **only** | deploy tasks skipped by tag |

| Вызов | Вайп? | Почему |
|-------|-------|--------|
| `deploy.yml` | Нет | переменная false, тег не запрошен |
| `deploy.yml --tags web_app_wipe` | Нет | его блокирует `when` (переменная всё ещё false) |
| `deploy.yml -e web_app_wipe=true` | Да — сначала вайп, **затем** деплой | чистая переустановка |
| `deploy.yml -e web_app_wipe=true --tags web_app_wipe` | Да — **только** вайп | задачи деплоя пропущены по тегу |

> Why two gates and not just the special `never` tag? `never` is a single gate — anyone who passes `--tags web_app_wipe` (or, with `never`, anyone who passes `--tags never`) triggers the wipe. A second, independent gate (the variable) means a typo or a CI mistake in **either** dimension alone is harmless: the other gate still blocks. Defence in depth, not cleverness in one place.
>
> Почему два гейта, а не просто специальный тег `never`? `never` — это один гейт: любой, кто передаст `--tags web_app_wipe` (или, с `never`, `--tags never`), запустит вайп. Второй независимый гейт (переменная) означает, что опечатка или ошибка в CI только в **одном** измерении безвредна: другой гейт всё ещё блокирует. Защита в глубину, а не хитрость в одном месте.

### 3.2 Implement

3.2 Реализация

**File:** `roles/web_app/tasks/wipe.yml`

**Файл:** `roles/web_app/tasks/wipe.yml`

The shape:

Форма:

```yaml
---
- name: Wipe web application
  block:
    - name: ___                                # YOUR TASK: name it — "compose down"
      ___:                                     # YOUR TASK: which module?
        project_src: "{{ compose_project_dir }}"
        state: ___                             # YOUR TASK: which state removes it?
      failed_when: ___                         # YOUR TASK: tolerate the already-clean case
                                               #            (hint: `false` is one answer; a list
                                               #             of conditions is the better one)

    - name: ___                                # YOUR TASK: remove the project dir
      ___:
        path: "___"
        state: ___

    - name: ___                                # YOUR TASK: small completion debug
      ___:
        msg: "___"
  when: ___ | bool                             # YOUR TASK: gate #1 — variable
                                               #            (why is `| bool` mandatory?
                                               #             see Common Pitfalls)
  tags: [___, ___]                             # YOUR TASK: gate #2 — opt-in tag
```

Пояснение: блок вайпа останавливает стек Compose, удаляет каталог проекта и выводит сообщение; запуск управляется переменной `web_app_wipe` и opt-in-тегом.

**File:** `roles/web_app/defaults/main.yml` — add:

**Файл:** `roles/web_app/defaults/main.yml` — добавь:

```yaml
web_app_wipe: ___                              # YOUR TASK: safe default — what should it be?
```

Пояснение: безопасное значение по умолчанию — `false`, чтобы вайп не срабатывал случайно.

The `include_tasks: wipe.yml` line from Task 2.3 sits **before** the deploy block, so `-e web_app_wipe=true` gives you wipe-then-deploy in one run.

Строка `include_tasks: wipe.yml` из задачи 2.3 стоит **перед** блоком деплоя, поэтому `-e web_app_wipe=true` даёт вайп с последующим деплоем за один запуск.

### 3.3 Verify all four rows

3.3 Проверь все четыре строки

`YOUR TASK`: run each of the four invocations in §3.1 and capture the output. For each, prove what you claim:

**Твоя задача:** выполни все четыре вызова из §3.1 и сохрани вывод. Для каждого докажи своё утверждение:

- Row 1 (plain `deploy.yml`) — search the recap or `--list-tasks` output for the wipe tasks; they should be skipped.
  * Строка 1 (обычный `deploy.yml`) — поищи wipe-задачи в recap или выводе `--list-tasks`; они должны быть пропущены.
- Row 2 (`--tags web_app_wipe` alone) — the wipe tasks **enter** the play but the `when:` skips them. The recap shows `skipped`, not `ok`.
  * Строка 2 (только `--tags web_app_wipe`) — wipe-задачи **попадают** в play, но `when:` их пропускает. В recap будет `skipped`, а не `ok`.
- Row 3 (`-e web_app_wipe=true`) — `docker ps` before (running) → `docker ps` after the wipe inside the run (gone) → after deploy (running again, fresh container ID).
  * Строка 3 (`-e web_app_wipe=true`) — `docker ps` до (контейнер работает) → `docker ps` после вайпа внутри запуска (контейнера нет) → после деплоя (снова работает, новый ID контейнера).
- Row 4 (`-e ... --tags web_app_wipe`) — wipe runs, deploy tasks skipped by tag, `docker ps` ends empty.
  * Строка 4 (`-e ... --tags web_app_wipe`) — вайп выполняется, задачи деплоя пропущены по тегу, `docker ps` в конце пуст.

### 3.4 Proof of work — paste into `docs/LAB06.md`

3.4 Доказательства работы — вставь в `docs/LAB06.md`

- Four terminal captures, one per row, each labelled with the invocation.
  * Четыре скриншота терминала, по одному на строку, каждый подписан вызовом.
- `docker ps --filter name=<app>` immediately before and immediately after the wipe-only run.
  * Вывод `docker ps --filter name=<app>` непосредственно до и сразу после запуска только с вайпом.
- A one-line note explaining **why `| bool` is mandatory** on the `when:`. (Spoiler in Common Pitfalls if you're stuck.)
  * Заметка в одну строку, объясняющая, **почему `| bool` обязателен** в `when:`. (Подсказка в «Частых ошибках», если застрял.)

---

## Task 4 — CI/CD with GitHub Actions (3 pts)

Задача 4 — CI/CD с GitHub Actions (3 балла)

Automate the whole thing: push to `main` → lint → deploy → verify. Prefer **OIDC short-lived credentials** to a long-lived SSH key where the target supports it.

Автоматизируй весь процесс: пуш в `main` → lint → deploy → verify. Где целевая среда поддерживает, предпочитай **короткоживущие учётные данные OIDC**, а не долгоживущий SSH-ключ.

### 4.1 Workflow skeleton

4.1 Каркас workflow

**File:** `.github/workflows/ansible-deploy.yml`

**Файл:** `.github/workflows/ansible-deploy.yml`

The shape — fill the bodies:

Форма — заполни тела:

```yaml
name: Ansible Deploy

on:
  push:
    branches: [main]
    paths:
      - '___'                                  # YOUR TASK: trigger on ansible/** changes
      - '!___'                                 # YOUR TASK: exclude docs-only changes
      - '___'                                  # YOUR TASK: self-test on workflow file edits
  workflow_dispatch:                           # manual deploys

permissions:
  contents: read
  id-token: ___                                # YOUR TASK: which value enables OIDC?

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install '___' ___             # YOUR TASK: pin ansible-core to 2.21.* + ansible-lint
      - run: ___                               # YOUR TASK: lint the ansible/ tree

  deploy:
    needs: ___                                 # YOUR TASK: depend on lint
    runs-on: ubuntu-24.04
    environment: ___                           # YOUR TASK: GH Environment name
                                               #            (gives you approval + per-env secrets)
    steps:
      - uses: actions/checkout@v4
      # YOUR TASK: pick ONE auth tier from §4.2 and run the playbook + smoke test
```

Пояснение: workflow состоит из job'ов `lint` и `deploy`; `lint` запускает ansible-lint, `deploy` зависит от него и выполняет плейбук в защищённом GitHub Environment, а фильтры путей ограничивают триггеры.

> Pin `ansible-core==2.21.*` so CI matches your local lockfile. ansible-core has **no LTS label** — pin the exact minor that matches your `community.docker` 4.x collection.
>
> Зафиксируй `ansible-core==2.21.*`, чтобы CI совпадал с локальным lock-файлом. У ansible-core **нет статуса LTS** — фиксируй точную минорную версию, совместимую с коллекцией `community.docker` 4.x.

### 4.2 Pick an auth tier

4.2 Выбери уровень аутентификации

> ⚠️ **First, the easy mistake:** the auto-injected `secrets.GITHUB_TOKEN` authenticates to **GitHub** (the API, GHCR). It is **not** an SSH key, it cannot log in to your VM. Plain SSH-to-VM always needs *something* — a key, or a federated mechanism that ultimately produces a key/session.
>
> ⚠️ **Сначала о типичной ошибке:** автоматически подставляемый `secrets.GITHUB_TOKEN` аутентифицируется в **GitHub** (API, GHCR). Это **не** SSH-ключ, им нельзя войти на твою VM. Обычному SSH до VM всегда нужно *что-то* — ключ или федеративный механизм, который в итоге выдаёт ключ/сессию.

> ⚠️ **OIDC for SSH-to-VM, honestly.** OIDC's win is "no long-lived secret in GitHub" — but the VM at the far end has to accept the federated identity. That means: AWS SSM Session Manager / EC2 Instance Connect, GCP IAP TCP-tunnel, or a cloud-provisioned short-lived SSH cert (e.g. Teleport, HashiCorp Boundary, Smallstep). **Raw `ssh user@ip` against a hand-built VM with `authorized_keys` does NOT speak OIDC.** Pick OIDC only if your Lab 4 cloud supports one of these paths; otherwise the honest answer is Tier 2 (Environment secret + a deploy key) and the lab grades you on **justifying the choice**, not on picking OIDC for cosmetics.
>
> ⚠️ **Честно про OIDC для SSH до VM.** Плюс OIDC — «никакого долгоживущего секрета в GitHub», но VM на другом конце должна принять федеративную идентичность. То есть нужен AWS SSM Session Manager / EC2 Instance Connect, GCP IAP TCP-tunnel или выданный облаком короткоживущий SSH-сертификат (например, Teleport, HashiCorp Boundary, Smallstep). **Голый `ssh user@ip` к собранной вручную VM с `authorized_keys` OIDC НЕ понимает.** Выбирай OIDC, только если облако из лабы 4 поддерживает один из этих путей; иначе честный ответ — уровень 2 (Environment secret + deploy-ключ), и лаба оценивает **обоснование выбора**, а не OIDC ради галочки.

| Tier | Mechanism | Trade-off |
|------|-----------|-----------|
| Repo secret as env | `secrets.SSH_KEY` → `~/.ssh/id_ed25519` | Simple; a long-lived key is the new "password in source" |
| GitHub Environment | Per-env secrets (deploy key per env) + manual approval | Needed for prod; small ops overhead |
| **OIDC + cloud IAM** | Short-lived federated token → cloud-mediated SSH (SSM / IAP / cert) | No static key in CI; requires the target to support a federated SSH path |

| Уровень | Механизм | Компромисс |
|---------|----------|------------|
| Секрет репозитория как env | `secrets.SSH_KEY` → `~/.ssh/id_ed25519` | Просто; долгоживущий ключ — это новый «пароль в исходниках» |
| GitHub Environment | Секреты на окружение (deploy-ключ на каждое окружение) + ручное подтверждение | Нужно для прода; небольшие операционные затраты |
| **OIDC + cloud IAM** | Короткоживущий федеративный токен → SSH через облако (SSM / IAP / сертификат) | Нет статического ключа в CI; требуется поддержка федеративного SSH на целевой стороне |

`YOUR TASK`: pick one, implement it. Justify your choice in the docs in 2–3 sentences referencing the trade-offs above. The skeletons:

**Твоя задача:** выбери один вариант и реализуй его. Обоснуй выбор в документации в 2–3 предложениях, ссылаясь на компромиссы выше. Каркасы:

**OIDC (recommended):**

**OIDC (рекомендуется):**

```yaml
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ___                  # YOUR TASK: an IAM role ARN trusting GH OIDC
          aws-region: ___
      - name: Deploy
        run: |
          ___                                  # YOUR TASK: ansible-playbook against your inventory
```

Пояснение: шаг `configure-aws-credentials` получает короткоживущие креды через IAM-роль, доверяющую OIDC GitHub, после чего плейбук запускается без статичных ключей.

**SSH-in-secret (fallback):**

**SSH-ключ в секрете (запасной вариант):**

```yaml
      - env:
          VAULT:  ${{ secrets.VAULT_PASS }}
          SSH:    ${{ secrets.SSH_KEY }}
          VM_IP:  ${{ secrets.VM_IP }}        # secret: target VM IP or DNS name from your Lab 4 / inventory
        run: |
          mkdir -p ~/.ssh && chmod 700 ~/.ssh
          install -m 600 /dev/stdin ~/.ssh/id_ed25519 <<< "$SSH"
          install -m 600 /dev/stdin /tmp/vault        <<< "$VAULT"
          ssh-keyscan -H "$VM_IP" >> ~/.ssh/known_hosts    # avoids host-key prompts
          ___                                  # YOUR TASK: ansible-playbook with --vault-password-file
```

Пояснение: секреты записываются в файлы через stdin с правами 600, а `ssh-keyscan` заранее добавляет host key, чтобы избежать интерактивных запросов.

> Never echo a secret to the log (no `run: echo "$SSH"` for debugging — pipe it into a file with redirected stdin, like the install above). A self-hosted runner on the target network removes the SSH key entirely — but it is a trust boundary, **never** run it on PRs from forks.
>
> Никогда не выводи секрет в лог (никаких `run: echo "$SSH"` для отладки — передавай его в файл через stdin с перенаправлением, как в установке выше). Self-hosted runner в целевой сети полностью убирает SSH-ключ — но это граница доверия: **никогда** не запускай его на PR из форков.

### 4.3 Verify in CI, not just exit 0

4.3 Проверяй в CI, а не только по коду выхода 0

`ansible-playbook` returning 0 only means the playbook ran without errors — it does **not** mean the app responds. Add a smoke-test step.

Код возврата 0 у `ansible-playbook` означает лишь, что плейбук выполнился без ошибок — это **не** значит, что приложение отвечает. Добавь шаг smoke-теста.

`YOUR TASK`: write a step that polls `/health` **on the deployed VM** (not `localhost` on the runner — that's just curling GitHub's runner) up to N times (you pick a bounded N), exits 0 on first 200, exits 1 if it never comes up. Use plain shell, not a third-party action. Hint: `curl -fsS http://<VM-IP>:<app_host_port>/health` returns non-zero on non-2xx; pair it with a small retry loop. Get the VM address from the same source the playbook used (inventory variable, GH variable, or `terraform output -json`).

**Твоя задача:** напиши шаг, который опрашивает `/health` **на развёрнутой VM** (а не `localhost` на раннере — это было бы просто обращение к раннеру GitHub) до N раз (N выбери конечным), завершается с кодом 0 при первом 200 и с кодом 1, если сервис так и не поднялся. Используй обычный shell, а не сторонний action. Подсказка: `curl -fsS http://<VM-IP>:<app_host_port>/health` возвращает ненулевой код при не-2xx; дополни его небольшим циклом повторов. Бери адрес VM из того же источника, что и play (переменная inventory, GH-переменная или `terraform output -json`).

Add the status badge to your repo `README.md`:

Добавь бейдж статуса в `README.md` своего репозитория:

```markdown
[![Ansible Deploy](https://github.com/<you>/<repo>/actions/workflows/ansible-deploy.yml/badge.svg)](https://github.com/<you>/<repo>/actions/workflows/ansible-deploy.yml)
```

### 4.4 Deployment strategy

4.4 Стратегия деплоя

`serial:` on a play caps how many hosts deploy **at the same moment**. It does **NOT** mean "deploy to 50% of hosts and stop." With 10 hosts and `serial: "50%"`, Ansible runs **two batches of 5** — every host eventually gets the new version. The lever controls *concurrency*, not *coverage*. For a true canary that stops to observe, use a list: `serial: [1, 5, "100%"]` — first batch is 1, then 5, then the rest.

Директива `serial:` в play ограничивает, сколько хостов деплоится **одновременно**. Она **НЕ** означает «задеплоить на 50% хостов и остановиться». При 10 хостах и `serial: "50%"` Ansible выполняет **две партии по 5** — новая версия в итоге доедет до каждого хоста. Рычаг управляет *параллелизмом*, а не *охватом*. Для настоящей канареечной выкатки с остановкой на наблюдение используй список: `serial: [1, 5, "100%"]` — сначала 1 хост, затем 5, затем остальные.

```yaml
- name: Deploy web tier
  hosts: webservers
  serial: "___"                                # YOUR TASK: pick one — explain the meaning
                                               #            in your docs (rolling vs canary vs all)
  max_fail_percentage: ___                     # YOUR TASK: pick a small int, justify
  roles: [web_app]
```

Пояснение: `serial` задаёт размер партии хостов, а `max_fail_percentage` останавливает выкатку, если доля упавших хостов превысит порог.

| Strategy | Typical `serial:` | When |
|----------|-------------------|------|
| Rolling | `1` or `25%` | Zero-downtime behind a load balancer |
| Canary | `1` (first batch, then observe) | Test on one host before continuing |
| All-at-once | omit `serial:` | Dev/staging only |

| Стратегия | Типичный `serial:` | Когда |
|-----------|-------------------|-------|
| Rolling | `1` или `25%` | Нулевой простой за балансировщиком |
| Canary | `1` (первая партия, затем наблюдение) | Проверка на одном хосте перед продолжением |
| All-at-once | не указывать `serial:` | Только dev/staging |

In your docs: which strategy did you pick for prod, and why?

В документации: какую стратегию ты выбрал для прода и почему?

### 4.5 Proof of work — paste into `docs/LAB06.md`

4.5 Доказательства работы — вставь в `docs/LAB06.md`

- A screenshot (or pasted log block) of a green workflow run.
  * Скриншот (или вставленный блок лога) успешного запуска workflow.
- Log lines showing **ansible-lint passing** (not skipped, not all-warnings).
  * Строки лога, показывающие **успешный проход ansible-lint** (не пропущен, не сплошные предупреждения).
- The smoke-test step output (curl 200).
  * Вывод шага smoke-теста (curl 200).
- The passing status badge URL.
  * URL бейджа со статусом «проходит».
- 2–3 sentences justifying your `serial:` + auth-tier choices.
  * 2–3 предложения с обоснованием выбора `serial:` и уровня аутентификации.

**Research:**

**Исследование:**

- Security implications: long-lived SSH key in a repo secret vs OIDC + cloud IAM. What's the blast radius if each leaks?
  * Вопросы безопасности: долгоживущий SSH-ключ в секрете репозитория против OIDC + cloud IAM. Каков радиус поражения при утечке каждого?
- How would you add a rollback job that triggers on the verify step failing? Sketch the YAML.
  * Как добавить job отката, который срабатывает при падении шага verify? Набросай YAML.
- Why must a self-hosted runner never execute jobs from fork PRs?
  * Почему self-hosted runner ни в коем случае не должен выполнять job'ы из PR форков?

---

## Bonus — Multi-Environment Promotion (2 pts)

Бонус — Продвижение между окружениями (2 балла)

> Single bonus task (2 pts). Promote **one immutable image** through `staging` → `prod` using per-environment inventories and a manual approval gate. Same artifact, never rebuilt, never retagged.
>
> Одно бонусное задание (2 балла). Продвигай **один неизменяемый образ** из `staging` → `prod`, используя отдельные inventory по окружениям и гейт ручного подтверждения. Тот же артефакт, без пересборки и без повторной разметки тегом.

Build on your Task 4 workflow. The whole point: the identical image (pinned by SHA or SemVer, **not** `:latest`) flows to staging automatically, then to prod only after a human approves.

Развивай workflow из задачи 4. Смысл в том, что идентичный образ (зафиксированный по SHA или SemVer, **не** `:latest`) автоматически попадает в staging, а в прод — только после ручного подтверждения.

`YOUR TASK`:

**Твоя задача:**

1. **Two inventories** with per-env pinned `docker_tag` (staging leads, prod lags). Both tags must point at the **same `docker_image`** that exists in your registry — promotion means "same image, newer environment", never a rebuild:
   * **Два inventory** с зафиксированным для каждого окружения `docker_tag` (staging впереди, prod отстаёт). Оба тега должны указывать на **один и тот же `docker_image`** из твоего реестра — продвижение означает «тот же образ, более новое окружение», а не пересборку:

   ```
   ansible/inventories/
     staging/hosts.ini   group_vars/all.yml   docker_tag: ___   # YOUR TASK: the newer SemVer or :sha-abc123 — what staging is testing
     prod/hosts.ini      group_vars/all.yml   docker_tag: ___   # YOUR TASK: the SemVer / SHA that already passed staging — what prod is running
   ```

   Пояснение: в каждом окружении свой `group_vars/all.yml` с зафиксированным `docker_tag`; образ берётся из реестра и не пересобирается.

   Per-env Vault files. Do **not** branch the playbook on `inventory_hostname == 'prod'` — branch on **variables only**.

   Файлы Vault для каждого окружения. **Не** ветви плейбук по `inventory_hostname == 'prod'` — ветвись **только по переменным**.

2. **Promotion workflow** — extend `ansible-deploy.yml` (or add `ansible-promote.yml`):
   * **Workflow продвижения** — расширь `ansible-deploy.yml` (или добавь `ansible-promote.yml`):
   - `deploy-staging` runs automatically on push to `main` against `inventories/staging`.
     * `deploy-staging` запускается автоматически при пуше в `main` в `inventories/staging`.
   - `deploy-prod` `needs: deploy-staging`, uses `environment: production` with a **required reviewer**, targets `inventories/prod`.
     * `deploy-prod` с `needs: deploy-staging` использует `environment: production` с **обязательным ревьюером** и целится в `inventories/prod`.
   - The **same** `docker_image` flows to both; only `docker_tag` differs per env.
     * В оба окружения идёт **один и тот же** `docker_image`; различается только `docker_tag` для каждого окружения.
   - Prod uses `serial:` + `max_fail_percentage:` from Task 4.4.
     * В проде используются `serial:` и `max_fail_percentage:` из задачи 4.4.

   Skeleton:

   Каркас:

   ```yaml
   deploy-prod:
     needs: ___                                # YOUR TASK
     environment: ___                          # YOUR TASK: env name with required reviewer
     runs-on: ubuntu-24.04
     steps:
       - uses: actions/checkout@v4
       # YOUR TASK: assume creds (auth tier from Task 4.2), then:
       #   - run deploy.yml -i ansible/inventories/prod
       #   - smoke-test /health on a prod host
   ```

   Пояснение: job `deploy-prod` зависит от стадии staging, требует Environment с обязательным ревьюером и целится в `inventories/prod`.

3. **Verify** end to end:
   * **Проверь** весь путь от начала до конца:
   - Staging deploy auto-runs on push.
     * Деплой staging запускается автоматически при пуше.
   - Prod job **pauses** for approval (screenshot the GitHub UI showing the gate).
     * Job прода **приостанавливается** до подтверждения (сделай скриншот интерфейса GitHub с гейтом).
   - Both envs serve the expected (different) tags via `/health` or a `?version=` field.
     * Оба окружения отдают ожидаемые (разные) теги через `/health` или поле `?version=`.

**Proof:** the two `group_vars/all.yml` files with different pinned tags, the prod approval gate screenshot, and `curl` output proving each env runs its pinned version.

**Доказательства:** два файла `group_vars/all.yml` с разными зафиксированными тегами, скриншот гейта подтверждения прода и вывод `curl`, доказывающий, что каждое окружение работает на своей зафиксированной версии.

---

## How to Submit

Как сдать работу

```bash
git switch -c lab06
git add ansible/ .github/workflows/ansible-deploy.yml ansible/docs/LAB06.md
git commit -m "feat(lab06): advanced ansible — blocks, tags, compose, CI"
git push -u origin lab06
```

Пояснение: создай ветку `lab06`, добавь нужные файлы, закоммить и запушь; перед коммитом проверь, что секреты не попали в индекс.

Confirm `.vault_pass` and any unencrypted secret are **not** staged. Encrypted Vault files are fine to commit.

Убедись, что `.vault_pass` и любые незашифрованные секреты **не** попали в индекс. Зашифрованные файлы Vault коммитить можно.

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab06` → `course-repo:master` *(reviewed)*
  * `your-fork:lab06` → `course-repo:master` *(с ревью)*
- `your-fork:lab06` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab06` → `your-fork:master` *(вливается в твой собственный main по завершении)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 — roles refactored, --list-tags + selective --tags captured, rescue fired
- [ ] Task 2 — Compose templated, idempotent (changed=0 on run #2), /health 200
- [ ] Task 3 — all four wipe rows verified, | bool justified
- [ ] Task 4 — workflow green, ansible-lint, smoke test, badge
- [ ] Bonus — two-env promotion behind required reviewer (optional)
```

---

## Acceptance Criteria

Критерии приёмки

### Main Tasks (10 pts)

Основные задачи (10 баллов)

**Blocks & Tags (2 pts):**

**Блоки и теги (2 балла):**

- ✅ `docker` and `common` refactored with `block`/`rescue`/`always`
  * ✅ `docker` и `common` переработаны с `block`/`rescue`/`always`
- ✅ Three-axis tag taxonomy applied to every block
  * ✅ Трёхосевая таксономия тегов применена к каждому блоку
- ✅ `--list-tags` output + one selective `--tags` run captured
  * ✅ Сохранён вывод `--list-tags` и один выборочный запуск с `--tags`
- ✅ Rescue path demonstrably **fired** (task output, not just `rescued=1`)
  * ✅ Путь rescue явно **сработал** (вывод задачи, а не только `rescued=1`)

**Docker Compose (3 pts):**

**Docker Compose (3 балла):**

- ✅ `app_deploy` renamed to `web_app`, all references updated
  * ✅ `app_deploy` переименована в `web_app`, все ссылки обновлены
- ✅ `meta/main.yml` declares the `docker` dependency
  * ✅ `meta/main.yml` объявляет зависимость от `docker`
- ✅ Jinja2 template uses interpolation **+** `{% if %}` **+** at least one filter
  * ✅ Шаблон Jinja2 использует интерполяцию **+** `{% if %}` **+** хотя бы один фильтр
- ✅ Deployed via `community.docker.docker_compose_v2`; both runs captured (changed → 0)
  * ✅ Деплой через `community.docker.docker_compose_v2`; сохранены оба запуска (changed → 0)
- ✅ App reachable on `/health`; container shows `(healthy)`
  * ✅ Приложение доступно на `/health`; контейнер показывает `(healthy)`

**Wipe Logic (2 pts):**

**Логика вайпа (2 балла):**

- ✅ Gated by `when: web_app_wipe | bool` **and** the `web_app_wipe` tag
  * ✅ Защищено `when: web_app_wipe | bool` **и** тегом `web_app_wipe`
- ✅ Included **before** the deploy block (clean reinstall works in one run)
  * ✅ Подключено **до** блока деплоя (чистая переустановка за один запуск)
- ✅ All four invocation rows captured
  * ✅ Сохранены все четыре строки вызовов

**CI/CD (3 pts):**

**CI/CD (3 балла):**

- ✅ Workflow runs `ansible-lint` then deploy
  * ✅ Workflow запускает `ansible-lint`, затем деплой
- ✅ Path filters configured (docs excluded; workflow self-test)
  * ✅ Настроены фильтры путей (документация исключена; самопроверка workflow)
- ✅ Auth via OIDC or GitHub Environment secret (no secret echoed)
  * ✅ Аутентификация через OIDC или секрет GitHub Environment (секрет не выводится)
- ✅ Smoke-test step + passing badge
  * ✅ Шаг smoke-теста + проходящий бейдж
- ✅ `serial:` chosen and justified
  * ✅ `serial:` выбран и обоснован

### Bonus (2 pts)

Бонус (2 балла)

- ✅ Separate `staging` / `prod` inventories with per-env pinned `docker_tag`
  * ✅ Отдельные inventory `staging` / `prod` с зафиксированным для окружения `docker_tag`
- ✅ Same immutable image promoted (no rebuild, no `:latest`)
  * ✅ Продвигается тот же неизменяемый образ (без пересборки, без `:latest`)
- ✅ Prod behind a required-reviewer Environment
  * ✅ Прод за Environment с обязательным ревьюером
- ✅ Rolling rollout with `serial:` + `max_fail_percentage:`
  * ✅ Rolling-выкатка с `serial:` + `max_fail_percentage:`
- ✅ Evidence both envs serve their pinned versions
  * ✅ Доказательства, что оба окружения отдают свои зафиксированные версии

---

## Rubric

Критерии оценивания

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Blocks & Tags | **2** | block/rescue/always; three-axis tags; rescue fired with output |
| **Task 2** — Docker Compose | **3** | renamed role + meta dep; Jinja2 (vars+if+filter); idempotent |
| **Task 3** — Wipe Logic | **2** | double-gated; all four rows verified |
| **Task 4** — CI/CD | **3** | lint → deploy → verify; OIDC/Environment auth; serial justified |
| **Bonus** — Multi-Env Promotion | **2** | immutable image; required-reviewer gate; both envs proven |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|--------|------:|----------|
| **Задача 1** — блоки и теги | **2** | block/rescue/always; трёхосевые теги; rescue сработал с выводом |
| **Задача 2** — Docker Compose | **3** | переименованная роль + meta-зависимость; Jinja2 (vars+if+filter); идемпотентность |
| **Задача 3** — логика вайпа | **2** | двойная защита; проверены все четыре строки |
| **Задача 4** — CI/CD | **3** | lint → deploy → verify; OIDC/Environment-аутентификация; serial обоснован |
| **Бонус** — продвижение между окружениями | **2** | неизменяемый образ; гейт обязательного ревьюера; оба окружения доказаны |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading scale:**

**Шкала оценивания:**

- **10/10:** Everything works; rescue + wipe + idempotency all proven; CI verifies; sharp docs.
  * **10/10:** всё работает; rescue, вайп и идемпотентность доказаны; CI проверяет; точная документация.
- **8–9/10:** All works; minor gaps in proof or docs.
  * **8–9/10:** всё работает; небольшие пробелы в доказательствах или документации.
- **6–7/10:** Core deploy works; weak tag strategy, missing rescue proof, or thin CI verify.
  * **6–7/10:** основной деплой работает; слабая стратегия тегов, нет доказательства rescue или поверхностная проверка в CI.
- **<6/10:** Missing Compose deploy, ungated wipe, or no working workflow.
  * **<6/10:** нет деплоя через Compose, вайп без защиты или неработающий workflow.

---

## Resources

Ресурсы

<details>
<summary>📚 Documentation</summary>

📚 Документация

- [Ansible Blocks (error handling)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
  * [Блоки Ansible (обработка ошибок)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
- [Ansible Tags](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html)
  * [Теги Ansible](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_tags.html)
- [Role dependencies](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#using-role-dependencies)
  * [Зависимости ролей](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#using-role-dependencies)
- [`community.docker.docker_compose_v2`](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
  * [`community.docker.docker_compose_v2`](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
- [Compose file reference](https://docs.docker.com/reference/compose-file/)
  * [Справочник по файлу Compose](https://docs.docker.com/reference/compose-file/)
- [Jinja2 templating](https://jinja.palletsprojects.com/)
  * [Шаблонизация Jinja2](https://jinja.palletsprojects.com/)
- [GHA `paths` filters](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-filters)
  * [Фильтры `paths` в GHA](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-filters)
- [GitHub OIDC for cloud deploys](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
  * [GitHub OIDC для облачных деплоев](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [Environments & required reviewers](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
  * [Environments и обязательные ревьюеры](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **Stale `/tmp/<project>` as a file blocks `state: directory`.** An aborted previous run can leave the project path as a file (or a symlink to one). The `file: state=directory` task fails with *"file exists, refusing to overwrite"*. Fix: `rm -rf <dir>` once, then re-run; or add an explicit `file: state=absent` before the create (only safe if your wipe doesn't depend on contents).
  * **Устаревший `/tmp/<project>` в виде файла блокирует `state: directory`.** Прерванный предыдущий запуск может оставить путь проекта файлом (или симлинком на файл). Задача `file: state=directory` падает с *«file exists, refusing to overwrite»*. Исправление: один раз выполни `rm -rf <dir>` и повтори запуск; либо добавь явный `file: state=absent` перед созданием (безопасно, только если вайп не зависит от содержимого).
- **Compose v2 vs v1 module names.** `community.docker.docker_compose` (v1, deprecated) and `community.docker.docker_compose_v2` (current) look one underscore apart. v1 shells out to the dead `docker-compose` Python CLI and errors with *"docker-compose: command not found"* on a fresh Ubuntu 24.04. **Always use `docker_compose_v2`.**
  * **Имена модулей Compose v2 и v1.** `community.docker.docker_compose` (v1, устарел) и `community.docker.docker_compose_v2` (актуальный) отличаются одним подчёркиванием. v1 вызывает мёртвый Python-CLI `docker-compose` и падает с *«docker-compose: command not found»* на чистой Ubuntu 24.04. **Всегда используй `docker_compose_v2`.**
- **FQCN vs bare module name.** `ansible-lint` flags `apt:` / `file:` / `template:` as `fqcn[action-core]`. Use `ansible.builtin.apt`, `ansible.builtin.file`, `ansible.builtin.template`. CI will fail your PR otherwise.
  * **FQCN против короткого имени модуля.** `ansible-lint` помечает `apt:` / `file:` / `template:` как `fqcn[action-core]`. Используй `ansible.builtin.apt`, `ansible.builtin.file`, `ansible.builtin.template`. Иначе CI завалит твой PR.
- **`serial: "50%"` semantics.** `serial` caps **concurrent** hosts, not total. With 10 hosts and `serial: "50%"`, Ansible runs **two batches of 5** — every host eventually gets the new version. It is **not** "deploy to 50% and stop." For a true canary, use `serial: [1, 5, "100%"]` (a list — first batch is 1, then 5, then the rest).
  * **Семантика `serial: "50%"`.** `serial` ограничивает **одновременное** число хостов, а не общее. При 10 хостах и `serial: "50%"` Ansible выполняет **две партии по 5** — новая версия в итоге доедет до всех. Это **не** «задеплоить на 50% и остановиться». Для настоящей канарейки используй `serial: [1, 5, "100%"]` (список — сначала 1, затем 5, затем остальные).
- **`-e web_app_wipe=true` is a STRING.** Ansible passes `--extra-vars key=value` as a Python `str`, not a `bool`. Without `| bool`, `when: web_app_wipe` evaluates the truthiness of the **string** — and *both* `"true"` and `"false"` are non-empty strings, so both pass. The gate that's supposed to keep prod safe fires on `-e web_app_wipe=false` too. The `| bool` filter coerces `"true"` / `"false"` / `"yes"` / `"no"` / `"1"` / `"0"` to real booleans. Forget it and the gate is broken **in the most dangerous direction** — wipe runs when you typed "false".
  * **`-e web_app_wipe=true` — это СТРОКА.** Ansible передаёт `--extra-vars key=value` как Python-`str`, а не `bool`. Без `| bool` выражение `when: web_app_wipe` проверяет истинность **строки** — при этом и `"true"`, и `"false"` — непустые строки, поэтому проходят обе. Гейт, который должен защищать прод, срабатывает и на `-e web_app_wipe=false`. Фильтр `| bool` приводит `"true"` / `"false"` / `"yes"` / `"no"` / `"1"` / `"0"` к настоящим булевым значениям. Забудешь — гейт ломается **в самую опасную сторону**: вайп срабатывает, когда ты написал «false».
- **`| default('latest')` in production.** Convenient for dev, fatal in prod — re-deploying with `docker_tag=latest` silently picks up whatever was pushed to the registry since. Pin per-env in `group_vars`. The Bonus task makes this explicit.
  * **`| default('latest')` в продакшене.** Удобно для dev, смертельно для прода — повторный деплой с `docker_tag=latest` молча подхватит всё, что успели запушить в реестр. Фиксируй тег для каждого окружения в `group_vars`. Бонусное задание делает это явным.
- **`become: true` repeated per task.** Lift it to the `block`; tagging the block also tags every task inside. Less noise, fewer drift bugs.
  * **`become: true`, повторённый в каждой задаче.** Подними его на `block`; тег на блоке также помечает все задачи внутри. Меньше шума, меньше ошибок из-за расхождений.
- **Lab 7 reuses port 8080.** Next lab's Compose stack defines an `app` service on host port 8080 too. If you keep Lab 6's stack running while you start Lab 7, the second `docker compose up -d` errors with *"port is already allocated"*. Pick a non-conflicting `app_host_port` here (e.g. 8084), or `docker compose down` Lab 6 before starting Lab 7. Lab 7 expects an `app=devops-python` container label — if you're keeping both alive, set that label on this lab's container too so Lab 7's relabel rule sees it.
  * **Лаба 7 тоже использует порт 8080.** Стек Compose из следующей лабы также определяет сервис `app` на хостовом порту 8080. Если оставить стек лабы 6 запущенным, когда начинаешь лабу 7, второй `docker compose up -d` упадёт с *«port is already allocated»*. Выбери здесь непересекающийся `app_host_port` (например, 8084) или сделай `docker compose down` для лабы 6 перед началом лабы 7. Лаба 7 ожидает метку контейнера `app=devops-python` — если держишь оба живыми, поставь эту метку и на контейнер этой лабы, чтобы правило переразметки в лабе 7 его увидело.

</details>

<details>
<summary>🛠️ Dev tools worth knowing</summary>

🛠️ Полезные инструменты разработчика

- `ansible-lint` — runs in CI; matches it locally first to save round-trips
  * `ansible-lint` — запускается в CI; прогоняй его локально первым, чтобы сэкономить итерации.
- `ansible-playbook --list-tasks --list-tags --list-hosts` — three flags, no execution; ideal for pre-flight checks
  * `ansible-playbook --list-tasks --list-tags --list-hosts` — три флага, без выполнения; идеально для проверок перед запуском.
- `ansible-doc -t filter -l` — every Jinja2 filter that ships with ansible-core
  * `ansible-doc -t filter -l` — все фильтры Jinja2, входящие в ansible-core.
- `jq` — pair with `community.docker.docker_compose_v2` `register:`-ed output
  * `jq` — пара к выводу `register:` из `community.docker.docker_compose_v2`.

</details>

---

## Looking Ahead

Что дальше

You've shipped the app — next labs see and measure it:

Ты выкатил приложение — следующие лабы увидят его и научатся измерять:

| Lab | What it adds to this service |
|---:|---|
| 7 | Loki + Alloy logging — deploy the stack with the very patterns from this lab |
| 8 | Prometheus `/metrics`, instrumented endpoints, RED-method PromQL |
| 9 | Kubernetes (k3d) — migrate from Compose to K8s, reusing `/health` |
| 10 | Helm 4 charts for templated K8s deploys |
| 13 | ArgoCD — declarative, pull-based GitOps |
| 14 | Argo Rollouts — canary + blue/green for the same service |

| Лаба | Что добавляет к этому сервису |
|-----:|-------------------------------|
| 7 | Логирование Loki + Alloy — разверни стек теми же паттернами, что и в этой лабе |
| 8 | Prometheus `/metrics`, инструментированные эндпоинты, PromQL по методу RED |
| 9 | Kubernetes (k3d) — переезд с Compose на K8s с переиспользованием `/health` |
| 10 | Helm 4-чарты для шаблонного деплоя в K8s |
| 13 | ArgoCD — декларативный pull-based GitOps |
| 14 | Argo Rollouts — канареечные + blue/green выкатки для того же сервиса |

---

**Good luck.** Block gives you the error boundary, tag gives you the scalpel, Compose gives you the declarative state, CI gives you the button — and the double gate keeps the button from going off by itself.

**Удачи.** Блок даёт границу ошибок, тег — скальпель, Compose — декларативное состояние, CI — кнопку, а двойной гейт не даёт кнопке нажаться самой.
