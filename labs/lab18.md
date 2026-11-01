# Lab 18 — Reproducible Builds with Nix

Лаба 18 — воспроизводимые сборки с Nix

![difficulty](https://img.shields.io/badge/difficulty-intermediate-yellow)
![topic](https://img.shields.io/badge/topic-Nix%20%26%20Reproducibility-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![type](https://img.shields.io/badge/type-Exam%20Alternative-purple)

> **Goal:** Build your Lab 1 service and a container image with Nix flakes, then prove two independent builds land on the **identical `/nix/store` path** — the bit-for-bit reproducibility that a `Dockerfile` cannot deliver.
> **Deliverable:** A PR from `lab18` adding `labs/lab18/` (your flake + sources) and `labs/submission18.md`. The PR description must include the store path you reproduced.
>
> **Цель:** собери сервис из лабы 1 и образ контейнера с помощью Nix-флейков, а затем докажи, что две независимые сборки попадают в **идентичный путь в `/nix/store`** — ту самую побитовую воспроизводимость, которую не может дать `Dockerfile`.
> **Что сдать:** PR из ветки `lab18`, добавляющий `labs/lab18/` (твой флейк + исходники) и `labs/submission18.md`. В описании PR обязательно укажи путь в хранилище, который ты воспроизвёл.

---

## Overview

Обзор

A `Dockerfile` and a `requirements.txt` *look* reproducible. They drift:

Файлы `Dockerfile` и `requirements.txt` *выглядят* воспроизводимыми. На деле они «плывут»:

- `FROM python:3.13-slim` points to different content over time — the tag is mutable
  * `FROM python:3.13-slim` со временем указывает на разное содержимое — тег изменяем
- `pip install -r requirements.txt` lets transitive deps move even with `==` pins on direct ones
  * `pip install -r requirements.txt` позволяет транзитивным зависимостям двигаться, даже если прямые закреплены через `==`
- Docker layers embed build timestamps, so the same `Dockerfile` produces a different image SHA every build
  * Слои Docker встраивают временные метки сборки, поэтому один и тот же `Dockerfile` при каждой сборке даёт разный SHA образа

Nix fixes this by treating the build as a **pure function of its inputs**. Every input — sources, dependencies, compiler, build flags — is hashed into the output path `/nix/store/<sha256>-<name>-<version>`. Same inputs → same hash → identical output, on any machine, today and in 2036. The same hash also means a cache hit on `cache.nixos.org` skips the rebuild entirely.

Nix решает это, рассматривая сборку как **чистую функцию от её входных данных**. Каждый вход — исходники, зависимости, компилятор, флаги сборки — хешируется в путь вывода `/nix/store/<sha256>-<name>-<version>`. Одинаковые входы → одинаковый хеш → идентичный результат на любой машине, сегодня и в 2036 году. Тот же хеш означает, что попадание в кеш на `cache.nixos.org` полностью избавляет от пересборки.

In this lab you will practice:

В этой лабе ты отработаешь:

- Writing a **flake** by hand — outer shape, inputs, outputs — choosing the right derivation builder for what you're packaging
  * Написание **флейка** вручную — внешняя структура, `inputs`, `outputs` — с выбором подходящего билдера derivation для того, что упаковываешь
- Pinning the full dependency closure with `flake.lock` (the anchor of reproducibility)
  * Закрепление всего замыкания зависимостей через `flake.lock` (якорь воспроизводимости)
- Building a container image with `dockerTools` — no `FROM`, no `apt-get`, no layer timestamps
  * Сборка образа контейнера через `dockerTools` — без `FROM`, без `apt-get`, без временных меток в слоях
- **Proving** two independent builds produce the **identical store path** — the bit-for-bit headline
  * **Доказательство** того, что две независимые сборки дают **идентичный путь в хранилище**, — главный побитовый результат лабы

> ⚠️ **Scope:** No Kubernetes, no CI. One service, one flake, two `nix build` runs, one hash comparison. Reproducibility is the whole point — don't get clever.
>
> ⚠️ **Границы лабы:** никакого Kubernetes и CI. Один сервис, один флейк, два запуска `nix build`, одно сравнение хешей. Воспроизводимость — и есть вся суть; не мудри.

**This is an Exam Alternative Lab.** Complete both Lab 17 (Cloudflare Workers) and Lab 18 (Nix) to the required bar to replace the final exam. See Lab 17 for the exam-alternative deadline and minimum-score rules. Taken on its own, Lab 18 is a **bonus** lab: **10 pts** of main tasks (6 + 4) + a **2 pt** bonus = **12 pts max**.

**Это лаба-альтернатива экзамену.** Чтобы заменить финальный экзамен, выполни обе лабы — 17 (Cloudflare Workers) и 18 (Nix) — на требуемом уровне. Сроки и правила минимального балла для альтернативы экзамену смотри в лабе 17. Сама по себе лаба 18 — **бонусная**: **10 баллов** за основные задачи (6 + 4) + **2 балла** бонуса = **максимум 12 баллов**.

**Tech stack (May 2026):** Nix **2.25+** (flakes stable since 2.18; Determinate / official installer) · `nixos/nix` container as the WSL-friendly path · nixpkgs `nixos-25.11` (release branch, pinned via flake input) · `dockerTools.buildLayeredImage` · Docker (to `docker load` the resulting image)

**Стек технологий (май 2026):** Nix **2.25+** (флейки стабильны с 2.18; Determinate / официальный установщик) · контейнер `nixos/nix` как удобный путь для WSL · nixpkgs `nixos-25.11` (релизная ветка, закреплённая через вход флейка) · `dockerTools.buildLayeredImage` · Docker (чтобы выполнить `docker load` полученного образа)

> **Lix:** a community fork of Nix (forked March 2024) is a drop-in `nix` replacement with faster evaluation and friendlier governance. Either Nix 2.25+ or current Lix works for this lab.
>
> **Lix:** форк Nix от сообщества (ответвлён в марте 2024) — drop-in замена `nix` с более быстрым вычислением и более дружелюбным управлением. Для этой лабы подойдёт любой Nix 2.25+ или актуальный Lix.

> **Note on outputs:** every command output, hash, and store path shown below is **illustrative** — yours will differ. What must be reproducible is that **your own** two builds produce the **identical** path.
>
> **О выводах:** все выводимые командами данные, хеши и пути в хранилище ниже — **иллюстративные**, у тебя они будут другими. Воспроизводимым должно быть другое: **твои собственные** две сборки дают **идентичный** путь.

---

## Project State

Состояние проекта

**You should have from previous labs:**

**Что должно остаться от предыдущих лаб:**

- `app_python/` — the Flask / FastAPI service from Lab 1 with `GET /` and `GET /health`
  * `app_python/` — сервис на Flask / FastAPI из лабы 1 с `GET /` и `GET /health`
- `app_python/Dockerfile` from Lab 2 (you'll contrast it against `dockerTools` in Task 2)
  * `app_python/Dockerfile` из лабы 2 (в задаче 2 ты сравнишь его с `dockerTools`)

> If you no longer have Lab 1/2 artifacts, a one-route Flask app (`app.py` with `/health`) plus a one-line `requirements.txt` is enough to complete every task. The skill being graded is *the flake*, not the app.
>
> Если артефактов лаб 1–2 больше нет, для выполнения всех задач хватит Flask-приложения с одним маршрутом (`app.py` с `/health`) и `requirements.txt` из одной строки. Оценивается *флейк*, а не приложение.

**This lab adds:**

**Эта лаба добавляет:**

- `labs/lab18/flake.nix` — your flake (Task 1)
  * `labs/lab18/flake.nix` — твой флейк (задача 1)
- `labs/lab18/flake.lock` — the lock file pinning the full closure (Task 1, committed)
  * `labs/lab18/flake.lock` — lock-файл, закрепляющий всё замыкание зависимостей (задача 1, коммитится)
- `labs/lab18/app/` — sources for your derivation
  * `labs/lab18/app/` — исходники для твоей derivation
- `labs/submission18.md` — your submission report
  * `labs/submission18.md` — твой отчёт по лабе

---

## Setup

Подготовка

Nix installs to `/nix` at the system root and needs `root` to do so. On WSL2 (and on locked-down hosts) the cleanest path is the **`nixos/nix` Docker image** — it ships Nix with flakes pre-enabled and leaves your host filesystem untouched. The reference submission was built that way.

Nix устанавливается в `/nix` в корне системы, и для этого нужен `root`. В WSL2 (и на ограниченных хостах) самый чистый путь — **Docker-образ `nixos/nix`**: он содержит Nix с уже включёнными флейками и не трогает файловую систему хоста. Эталонное решение было собрано именно так.

**Path A — `nixos/nix` container (recommended on WSL / locked-down hosts):**

**Путь A — контейнер `nixos/nix` (рекомендуется в WSL / на ограниченных хостах):**

```bash
# Pinned image tag — do NOT use :latest (mutable). 2.34 or any 2.25+ is fine.
docker run --rm -it \
  -v "$PWD":/work -w /work \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --entrypoint /bin/sh \
  nixos/nix:2.34.0

# inside the container — flakes need the experimental feature flag explicit
nix --version
nix --extra-experimental-features 'nix-command flakes' eval --expr '1 + 1'
```

Пояснение: `docker run` поднимает одноразовый контейнер, монтирует текущий каталог в `/work` и передаёт сокет Docker, а внутри проверяется версия Nix и включаются экспериментальные команды `nix-command` и `flakes`.

The `docker.sock` mount lets `docker load < result` work from inside the Nix container in Task 2.

Монтирование `docker.sock` позволяет выполнить `docker load < result` изнутри Nix-контейнера в задаче 2.

**Path B — host install (Linux / macOS with sudo):**

**Путь B — установка на хост (Linux / macOS с sudo):**

```bash
# Determinate Systems installer — flakes on by default, clean uninstall
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s -- install
nix --version            # expect 2.25.x or newer
```

Пояснение: установщик Determinate Systems включает флейки по умолчанию и так же чисто удаляется.

<details>
<summary>🐧 Alternative: official Nix installer — use this if `install.determinate.systems` is unreachable or times out from your network (flakes must be enabled by hand)</summary>

🐧 Альтернатива: официальный установщик Nix — используй его, если `install.determinate.systems` недоступен или отваливается по таймауту из твоей сети (флейки придётся включить вручную)

```bash
sh <(curl -L https://nixos.org/nix/install) --daemon
```

Then add to `~/.config/nix/nix.conf` and restart your terminal:

Затем добавь в `~/.config/nix/nix.conf` и перезапусти терминал:

```
experimental-features = nix-command flakes
```

</details>

Sanity check (either path) — run a package without installing it:

Проверка работоспособности (любой путь) — запусти пакет, не устанавливая его:

```bash
nix run nixpkgs#hello   # downloads + runs `hello`, installs nothing permanent
```

Пояснение: `nix run` скачивает пакет во временное окружение, запускает его и ничего не оставляет в системе насовсем.

Create the layout:

Создай структуру:

```
labs/lab18/
├── flake.nix          # YOU WRITE — Task 1
├── flake.lock         # generated by `nix flake update`, committed
├── app/               # your Lab 1 sources (or the minimal stub)
│   ├── app.py
│   └── requirements.txt
└── (submission18.md lives one level up at labs/submission18.md)
```

Пояснение: `flake.nix` ты пишешь сам, `flake.lock` генерируется и коммитится, а `app/` содержит исходники приложения из лабы 1 или минимальную заглушку.

---

## Task 1 — Write a flake that builds the Lab 1 app (6 pts)

Задача 1 — напиши флейк, который собирает приложение из лабы 1 (6 баллов)

**Objective:** Hand-write `flake.nix` from a structural skeleton, build the Lab 1 service through it, and commit `flake.lock` so the closure is pinned.

**Цель:** вручную написать `flake.nix` по структурному скелету, собрать через него сервис из лабы 1 и закоммитить `flake.lock`, чтобы замыкание зависимостей было закреплено.

### 1.1 — Outer shape of the flake

1.1 — Внешняя структура флейка

A flake is a Nix expression with three top-level fields: `description`, `inputs`, `outputs`. **What's inside each is yours.** The skeleton below shows only the **shape** — the choices that make this *your* reproducible build are blank.

Флейк — это выражение Nix с тремя полями верхнего уровня: `description`, `inputs`, `outputs`. **Что внутри каждого — решаешь ты.** Скелет ниже показывает только **форму** — выборы, которые делают эту сборку *твоей* воспроизводимой сборкой, оставлены пустыми.

```nix
# labs/lab18/flake.nix
{
  description = ___;                       # YOUR TASK: one short string describing what this flake builds

  inputs = {
    # YOUR TASK: pick a NIXPKGS REVISION AND PIN IT.
    # The lab tests `nixos-25.11` (the Nov-2024 release branch). NOT `nixpkgs-unstable`
    # — unstable means non-reproducible across days. Use a release tag.
    # Format: github:NixOS/nixpkgs/<branch-or-rev>
    nixpkgs.url = ___;
  };

  outputs = { self, nixpkgs }:
    let
      # YOUR TASK: pick the system you're building for.
      #   x86_64-linux  — WSL2 / most Linux laptops / CI
      #   aarch64-linux — Linux on Apple Silicon under Docker / Asahi
      #   aarch64-darwin — Apple Silicon macOS native
      #   x86_64-darwin  — Intel macOS
      system = ___;
      pkgs   = ___;                        # YOUR TASK: legacyPackages for the chosen system
    in {
      packages.${system}.default = ___;    # YOUR TASK: a derivation that builds your app — see 1.2
      # Task 2 will add a second output here: packages.${system}.dockerImg
    };
}
```

Пояснение: в скелете `___` — места, которые нужно заполнить; комментарии подсказывают, что именно выбрать, но не дают готового ответа.

**Why each line is a rubric line:**

**Почему каждая строка важна для оценки:**

- **`description`** — one sentence; appears in `nix flake show` and `nix flake metadata`. Make it useful.
  * **`description`** — одно предложение; появляется в `nix flake show` и `nix flake metadata`. Сделай его полезным.
- **`inputs.nixpkgs.url`** — *the* reproducibility anchor. A release tag (`nixos-25.11`) or a specific commit SHA freezes the entire dependency tree. **`nixpkgs-unstable` is wrong here** — its tip moves daily and is the most common "reproducible build that isn't" mistake.
  * **`inputs.nixpkgs.url`** — *тот самый* якорь воспроизводимости. Релизный тег (`nixos-25.11`) или конкретный SHA коммита замораживает всё дерево зависимостей. **`nixpkgs-unstable` здесь — ошибка**: его вершина движется каждый день, и это самая частая ошибка «воспроизводимой сборки, которая таковой не является».
- **`system`** — flake outputs are per-system. Pick the one you'll demo on; classmates on other systems can add their own to your flake.
  * **`system`** — выходы флейка задаются для каждой системы отдельно. Выбери ту, на которой будешь демонстрировать; однокурсники на других системах смогут добавить свои в твой флейк.
- **`pkgs`** — `nixpkgs.legacyPackages.${system}` is the canonical handle for "the package set, evaluated for this system." Memorise it.
  * **`pkgs`** — `nixpkgs.legacyPackages.${system}` — канонический способ получить «набор пакетов, вычисленный для этой системы». Запомни его.
- **`packages.${system}.default`** — what `nix build` (no `.#attr`) builds. This is your service.
  * **`packages.${system}.default`** — то, что собирает `nix build` (без `.#attr`). Это твой сервис.

### 1.2 — Pick a derivation and fill its body

1.2 — Выбери derivation и заполни её тело

The Nix store has dozens of builder functions (`mkDerivation`, `writeShellApplication`, `buildPythonApplication`, `buildGoModule`, …). Each one makes different assumptions about what it's packaging. Pick the one that matches your Lab 1 stack:

В хранилище Nix есть десятки функций-билдеров (`mkDerivation`, `writeShellApplication`, `buildPythonApplication`, `buildGoModule`, …). Каждая делает свои предположения о том, что упаковывает. Выбери ту, что соответствует стеку твоей лабы 1:

| Builder | Best for | Key inputs |
|---|---|---|
| `pkgs.writeShellApplication` | A wrapper script that calls something else (the reference submission uses this) | `name`, `runtimeInputs`, `text` |
| `pkgs.stdenv.mkDerivation` | Generic — you supply `buildPhase` / `installPhase` | `src`, `buildInputs`, `installPhase` |
| `pkgs.python3Packages.buildPythonApplication` | A Python app with `pyproject.toml` / `setup.py` | `pname`, `version`, `src`, `propagatedBuildInputs`, `format` |
| `pkgs.buildGoModule` | A Go program (Lab 1 bonus) | `pname`, `src`, `vendorHash` |

| Билдер | Для чего лучше всего | Ключевые входы |
|---|---|---|
| `pkgs.writeShellApplication` | Скрипт-обёртка, который вызывает что-то ещё (эталонное решение использует именно его) | `name`, `runtimeInputs`, `text` |
| `pkgs.stdenv.mkDerivation` | Универсальный — `buildPhase` / `installPhase` задаёшь сам | `src`, `buildInputs`, `installPhase` |
| `pkgs.python3Packages.buildPythonApplication` | Python-приложение с `pyproject.toml` / `setup.py` | `pname`, `version`, `src`, `propagatedBuildInputs`, `format` |
| `pkgs.buildGoModule` | Go-программа (бонус лабы 1) | `pname`, `src`, `vendorHash` |

**`YOUR TASK`**: in the `default = ___` blank from 1.1, call the builder that fits your app. Examples of the **shape only** (the inside is still yours):

**ТВОЯ ЗАДАЧА**: в пропуске `default = ___` из 1.1 вызови билдер, подходящий твоему приложению. Примеры показывают **только форму** (внутри по-прежнему твои решения):

```nix
# Shape 1 — writeShellApplication (good if your "app" is a thin shell entrypoint
# that calls into python/curl/whatever from the pinned closure)
packages.${system}.default = pkgs.writeShellApplication {
  name         = ___;                       # YOUR TASK: the binary name produced under result/bin/
  runtimeInputs = with pkgs; [ ___ ];       # YOUR TASK: every command your text uses (python3, curl, …)
  text         = ___;                       # YOUR TASK: the shell body. Single-quoted ''…'' multiline OK
};
```

```nix
# Shape 2 — buildPythonApplication (if you have a real Python package)
packages.${system}.default = pkgs.python3Packages.buildPythonApplication {
  pname   = ___;                            # YOUR TASK
  version = ___;                            # YOUR TASK
  src     = ___;                            # YOUR TASK — usually ./app
  format  = ___;                            # YOUR TASK — "other" if no pyproject/setup
  propagatedBuildInputs = with pkgs.python3Packages; [ ___ ]; # YOUR TASK — flask, etc.
  nativeBuildInputs     = [ pkgs.makeWrapper ];
  installPhase = ___;                       # YOUR TASK — copy app.py → $out/bin, wrapProgram with PYTHONPATH
};
```

```nix
# Shape 3 — buildGoModule (if you did the Lab 1 Go bonus)
packages.${system}.default = pkgs.buildGoModule {
  pname      = ___;                         # YOUR TASK
  version    = ___;                         # YOUR TASK
  src        = ___;                         # YOUR TASK
  vendorHash = ___;                         # YOUR TASK — null if no deps; else a sha256 from lib.fakeHash discovery
};
```

Пояснение: три скелета показывают форму вызова трёх разных билдеров; какой выбрать — зависит от того, что именно ты упаковываешь.

> **Why no PyPI?** A Nix build is **sandboxed with no network** by default. Nix pulls packages from the pinned nixpkgs revision (`Flask` → `pkgs.python3Packages.flask`). That's *the* reason the closure is reproducible: every dep is itself a fixed `/nix/store/...` path, not a `pip resolve` away.
>
> **Почему без PyPI?** Сборка Nix по умолчанию выполняется **в песочнице без сети**. Пакеты Nix берёт из закреплённой ревизии nixpkgs (`Flask` → `pkgs.python3Packages.flask`). Именно поэтому замыкание воспроизводимо: каждая зависимость — сама по себе фиксированный путь `/nix/store/...`, а не результат `pip resolve`.

<details>
<summary>📚 Where to learn the derivation syntax</summary>

📚 Где изучить синтаксис derivation

- [nix.dev — Building and running Python apps](https://nix.dev/tutorials/nixos/building-and-running-python-apps)
  * [nix.dev — сборка и запуск Python-приложений](https://nix.dev/tutorials/nixos/building-and-running-python-apps)
- [nixpkgs Python manual](https://nixos.org/manual/nixpkgs/stable/#python)
  * [Руководство по Python в nixpkgs](https://nixos.org/manual/nixpkgs/stable/#python)
- [`writeShellApplication`](https://nixos.org/manual/nixpkgs/stable/#trivial-builder-writeShellApplication) — the lightest builder, shellcheck-validated
  * [`writeShellApplication`](https://nixos.org/manual/nixpkgs/stable/#trivial-builder-writeShellApplication) — самый лёгкий билдер, проверяется shellcheck
- [Nix Pills — Our first derivation](https://nixos.org/guides/nix-pills/our-first-derivation.html)
  * [Nix Pills — наша первая derivation](https://nixos.org/guides/nix-pills/our-first-derivation.html)

Pick **one** builder and defend the choice in `submission18.md` — TA grading docks for builder choices that don't match what's actually being packaged.

Выбери **один** билдер и обоснуй выбор в `submission18.md` — преподаватель снижает балл за билдер, не соответствующий тому, что реально упаковывается.

</details>

### 1.3 — Generate `flake.lock` and commit it

1.3 — Сгенерируй `flake.lock` и закоммить его

`flake.lock` records the exact git SHA of every flake input. Without it, `nixpkgs/nixos-25.11` resolves to *whatever the branch tip is today* — fine for a few weeks, drifting silently over months. The lock file is what makes "build in 2026" identical to "build in 2036".

`flake.lock` записывает точный git SHA каждого входа флейка. Без него `nixpkgs/nixos-25.11` разрешается в *то, где ветка находится сегодня*, — это нормально на пару недель, но месяцами незаметно дрейфует. Именно lock-файл делает сборку 2026 года идентичной сборке 2036-го.

```bash
cd labs/lab18
git add flake.nix app/                    # flakes only see git-tracked files — see Pitfalls
nix flake update                           # writes flake.lock; pins each input to a specific rev
git add flake.lock                         # MUST be committed
nix build                                  # builds packages.${system}.default
./result/bin/<your-name>                   # runs your app
```

Пояснение: файлы должны быть добавлены в git до сборки, потому что флейки видят только отслеживаемые файлы; `nix flake update` записывает `flake.lock`, а `nix build` собирает пакет по умолчанию.

`YOUR TASK`: open `flake.lock` and **answer in `submission18.md`**:

**ТВОЯ ЗАДАЧА**: открой `flake.lock` и **ответь в `submission18.md`**:

- What field pins the exact `nixpkgs` revision? (Hint: it's not `ref`. Look at `locked.rev` and `locked.narHash`.)
  * Какое поле закрепляет точную ревизию `nixpkgs`? (Подсказка: это не `ref`. Смотри на `locked.rev` и `locked.narHash`.)
- Why does committing `flake.lock` matter, given you already pinned `nixpkgs-24.11` in `flake.nix`?
  * Почему коммит `flake.lock` важен, если ты уже закрепил `nixpkgs-24.11` в `flake.nix`?

### 1.4 — Proof of work

1.4 — Подтверждение работы

**Paste into `labs/submission18.md`:**

**Вставь в `labs/submission18.md`:**

- Your `flake.nix` contents (it's short — paste the whole thing) with a one-line note per `___` you filled in
  * Содержимое твоего `flake.nix` (он короткий — вставь целиком) с однострочной заметкой к каждому `___`, который ты заполнил
- The first 20 lines of your `flake.lock` (specifically, the `nixpkgs` node showing `locked.rev` + `locked.narHash`)
  * Первые 20 строк твоего `flake.lock` (а именно узел `nixpkgs` с `locked.rev` + `locked.narHash`)
- The output of:
  * Вывод команд:

  ```bash
  nix build .#default
  readlink result                            # /nix/store/<hash>-<name>  ← capture this exact path
  ./result/bin/<your-name>                   # proves the binary runs
  ```

- Your one-paragraph answer to 1.3's "what does the lock pin?" question
  * Твой ответ одним абзацем на вопрос из 1.3: «что именно закрепляет lock?»

---

## Task 2 — Reproducible Docker image with `dockerTools` (4 pts)

Задача 2 — воспроизводимый Docker-образ с `dockerTools` (4 балла)

**Objective:** Build a container image **from your flake** using `dockerTools.buildLayeredImage`, then contrast it with your Lab 2 Dockerfile (whose hash changes every build).

**Цель:** собрать образ контейнера **из твоего флейка** через `dockerTools.buildLayeredImage`, а затем сравнить его с Dockerfile из лабы 2 (чей хеш меняется при каждой сборке).

### 2.1 — Show your Lab 2 image is not reproducible

2.1 — Покажи, что образ из лабы 2 невоспроизводим

```bash
docker build -t lab2-app:v1 ./app_python && docker save lab2-app:v1 | sha256sum
docker build -t lab2-app:v2 ./app_python && docker save lab2-app:v2 | sha256sum
```

Пояснение: дважды собирается один и тот же Dockerfile, образы сохраняются в tar и хешируются — суммы должны различаться.

`YOUR TASK`: capture both SHA256 sums and confirm they differ. They do — Docker writes build timestamps into the layers, and `python:3.13-slim` can move under you between the two builds. Paste both sums into `submission18.md`.

**ТВОЯ ЗАДАЧА**: зафиксируй обе суммы SHA256 и убедись, что они различаются. Так и будет: Docker записывает временные метки сборки в слои, а `python:3.13-slim` может незаметно измениться между двумя сборками. Вставь обе суммы в `submission18.md`.

### 2.2 — Add a `dockerImg` output to your flake

2.2 — Добавь в флейк вывод `dockerImg`

Add a second package output to the flake from Task 1. The skeleton below shows the **shape** of `buildLayeredImage` — every body field is yours. There is **no fully-formed example** — by the time you see one whole, it's typing practice, not learning.

Добавь второй пакетный вывод в флейк из задачи 1. Скелет ниже показывает **форму** `buildLayeredImage` — все поля тела твои. **Полностью готового примера нет**: к тому моменту, как ты увидишь целый пример, это будет упражнение на печатание, а не обучение.

```nix
# Inside the same outputs = { … } block from Task 1, alongside packages.${system}.default
packages.${system}.dockerImg = pkgs.dockerTools.buildLayeredImage {
  name = ___;          # YOUR TASK: the image name (becomes `docker images` REPOSITORY).
                       # Not "latest" — that's a tag, not a name. Pick something self-explanatory.

  tag  = ___;          # YOUR TASK: a SemVer / CalVer / git-SHA tag. NOT "latest" (slide 20 of Lec 2).

  contents = [ ___ ];  # YOUR TASK: what goes inside the image filesystem.
                       # At minimum, your Task-1 package (referenced by `self.packages.${system}.default`
                       # OR a local `let app = … in app` binding). Add pkgs.coreutils / pkgs.cacert
                       # only if your binary needs `ls`/`cat` or HTTPS at runtime.

  config = {
    Cmd          = [ ___ ];          # YOUR TASK: the command that runs at container start. Exec form.
                                     # Point at the binary your Task-1 derivation produced —
                                     # e.g. "${app}/bin/<name>". Shell form would re-introduce the PID-1 trap
                                     # you learned to avoid in Lab 2.
    ExposedPorts = { ___ = {}; };    # YOUR TASK: the port your service listens on (Lab 1 default: 5000/tcp;
                                     # Lab 2's container overrode this via PORT=8080 env var, but a fresh Nix
                                     # build with writeShellApplication honors Lab 1's PORT default).
                                     # The key is a string like "5000/tcp". If you want 8080, also add
                                     # `Env = [ "PORT=8080" ];` to this config block.
  };

  # THE reproducibility trick. Nix gives you a knob most container builders don't:
  # the embedded creation timestamp. Set it to a FIXED epoch instead of "now".
  created = ___;       # YOUR TASK: a fixed RFC-3339 timestamp string. NOT "now" — that re-introduces
                       # non-determinism (every build differs). NOT 1970-01-01T00:00:00Z either
                       # (Docker tooling sometimes rejects the literal Unix epoch). The accepted convention
                       # is `1970-01-01T00:00:01Z` — one second after the epoch.
};
```

Пояснение: `name` и `tag` задают имя и тег образа; `contents` — что попадёт в файловую систему; `Cmd` — команда запуска в exec-форме; `ExposedPorts` — открытый порт; `created` — фиксированный момент создания вместо `now`.

> **`buildLayeredImage`** produces an efficient, content-addressed layered tarball — no `FROM` base image, no `apt-get`, no `RUN`. The pinned `created` epoch is what removes the last source of non-determinism.
>
> **`buildLayeredImage`** создаёт эффективный слоистый tar-архив с адресацией по содержимому — без базового образа `FROM`, без `apt-get`, без `RUN`. Закреплённый `created` — это то, что убирает последний источник недетерминизма.

<details>
<summary>📚 Where to learn dockerTools</summary>

📚 Где изучить dockerTools

- [nix.dev — Building and running Docker images](https://nix.dev/tutorials/nixos/building-and-running-docker-images.html)
  * [nix.dev — сборка и запуск Docker-образов](https://nix.dev/tutorials/nixos/building-and-running-docker-images.html)
- [nixpkgs `dockerTools` reference](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools) — `buildImage` vs `buildLayeredImage` vs `streamLayeredImage`
  * [Справочник по `dockerTools` в nixpkgs](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools) — `buildImage` vs `buildLayeredImage` vs `streamLayeredImage`
- [pkgs.cacert](https://search.nixos.org/packages?show=cacert) — if your app calls HTTPS
  * [pkgs.cacert](https://search.nixos.org/packages?show=cacert) — если твоё приложение ходит по HTTPS

</details>

### 2.3 — Build, load, run

2.3 — Собери, загрузи, запусти

```bash
nix build .#dockerImg                  # → result is an image tarball, not a symlink to a binary
file result                            # confirms it's a gzip'd tar

# From inside the Nix container, the docker.sock mount lets this work; on host, you have docker directly
docker load < result                   # loads <your-name>:<your-tag>
docker run -d -p 5001:5000 --name lab18 <your-name>:<your-tag>      # if you exposed 5000/tcp
# or, if you set Env = [ "PORT=8080" ] AND ExposedPorts = { "8080/tcp" = {}; }:
# docker run -d -p 5001:8080 --name lab18 <your-name>:<your-tag>
curl http://localhost:5001/health      # behaves like the Lab 2 container
```

Пояснение: `nix build .#dockerImg` создаёт tar-архив образа, `docker load` загружает его в Docker, после чего контейнер запускается и проверяется через `curl`.

### 2.4 — Compare against the Lab 2 image

2.4 — Сравни с образом из лабы 2

```bash
docker images --format '{{.Repository}}\t{{.Tag}}\t{{.Size}}' | grep -E 'lab2-app|<your-name>'
docker history <your-name>:<your-tag>
```

`YOUR TASK`: read your own `docker history`. The Nix image typically:

**ТВОЯ ЗАДАЧА**: посмотри собственный вывод `docker history`. Образ из Nix обычно:

- Has **layers whose hashes are content-addressed** (same content → same layer hash, across builds and machines)
  * Имеет **слои, хеши которых адресуются по содержимому** (одинаковое содержимое → одинаковый хеш слоя, между сборками и машинами)
- Ships **only the closure** — no Debian base, no `apt` cache, often smaller than the slim-based image
  * Содержит **только замыкание зависимостей** — без базы Debian, без кеша `apt`, часто меньше образа на slim-базе
- Has a `CREATED` timestamp that's **the epoch you set**, not `now`
  * Имеет метку `CREATED`, равную **заданной тобой эпохе**, а не `now`

In `submission18.md`, write a one-paragraph finding comparing your Lab 2 image to the `dockerTools` image: size, layer count, and the `CREATED` column.

В `submission18.md` напиши вывод одним абзацем, сравнив свой образ из лабы 2 с образом `dockerTools`: размер, число слоёв и колонку `CREATED`.

### 2.5 — Proof of work

2.5 — Подтверждение работы

**Paste into `labs/submission18.md`:**

**Вставь в `labs/submission18.md`:**

- The `dockerImg` block of your `flake.nix` (the contents + config you wrote)
  * Блок `dockerImg` из твоего `flake.nix` (написанные тобой `contents` + `config`)
- The two **differing** `docker save | sha256sum` outputs from 2.1
  * Два **различающихся** вывода `docker save | sha256sum` из 2.1
- The `docker load` log line showing the loaded image name + tag
  * Строку лога `docker load` с именем и тегом загруженного образа
- A real `curl` against `/health` proving the loaded image runs
  * Реальный `curl` к `/health`, доказывающий, что загруженный образ работает
- The `docker images` + `docker history` rows comparing Lab 2 vs your Nix image
  * Строки `docker images` + `docker history`, сравнивающие лабу 2 и твой образ из Nix
- A two-sentence analysis: *why* a plain `Dockerfile` can't reach bit-for-bit reproducibility (the epoch trick, the base-image drift, the `apt-get` non-determinism). Reference the lecture, slides 15 and 18.
  * Анализ на два предложения: *почему* обычный `Dockerfile` не может достичь побитовой воспроизводимости (трюк с эпохой, дрейф базового образа, недетерминизм `apt-get`). Сославшись на лекцию, слайды 15 и 18.

---

## Bonus Task — Prove the build is bit-for-bit reproducible (2 pts)

Бонусная задача — докажи побитовую воспроизводимость сборки (2 балла)

**Objective:** Run **two** independent `nix build` invocations and show they land on the **identical `/nix/store` path**. This is the headline of the entire lab — a successful build by itself isn't proof; *two builds that match* is.

**Цель:** выполнить **два** независимых вызова `nix build` и показать, что они попадают в **идентичный путь в `/nix/store`**. Это главный результат всей лабы — успешная сборка сама по себе не доказательство; *доказательство — две совпадающие сборки*.

The reference submission's proof is **two `nix build .#default` runs, the second with `--rebuild`, both producing the same store path** plus an explicit hash equality. Nix's `--rebuild` flag forces a real rebuild (not a cache hit) and then **verifies that the output of the new build matches the existing store path bit-for-bit** — if anything differed, Nix would report it loudly.

Доказательство в эталонном решении — **два запуска `nix build .#default`, второй с `--rebuild`, оба дают один и тот же путь в хранилище**, плюс явное равенство хешей. Флаг `--rebuild` заставляет Nix выполнить настоящую пересборку (а не взять из кеша), а затем **проверяет, что результат новой сборки побитово совпадает с существующим путём в хранилище** — любое расхождение Nix громко сообщил бы.

`YOUR TASK`: design and execute the proof yourself. The lab does NOT give you the commands. You will need to:

**ТВОЯ ЗАДАЧА**: придумай и выполни доказательство сам. Лаба НЕ даёт готовых команд. Тебе нужно:

1. Build `packages.${system}.default` once, capture the resulting store path
   * Собрать `packages.${system}.default` один раз и зафиксировать полученный путь в хранилище
2. Force Nix to do a **real, full rebuild** (not a cache hit) and capture the path the rebuild lands on
   * Заставить Nix выполнить **настоящую полную пересборку** (а не взять из кеша) и зафиксировать путь, в который она попадёт
3. Show that both paths are **identical** — and that Nix itself validates this in step 2 ("identical store path both builds" / "checking outputs of …")
   * Показать, что оба пути **идентичны**, и что Nix сам подтверждает это на шаге 2 («identical store path both builds» / «checking outputs of …»)
4. Repeat the same proof for `packages.${system}.dockerImg` (the image tarball) — `sha256sum result` from two clean builds must match
   * Повторить то же доказательство для `packages.${system}.dockerImg` (tar-архив образа) — `sha256sum result` из двух чистых сборок должен совпасть
5. Explain in `submission18.md` *why* this works: the role of the input hash, why `flake.lock` is the precondition, why `--rebuild` is the right command (`nix-store --delete` alone is brittle)
   * Объяснить в `submission18.md`, *почему* это работает: роль хеша входных данных, почему `flake.lock` — предварительное условие, почему `--rebuild` — правильная команда (`nix-store --delete` сам по себе ненадёжен)

Hints (not commands):

Подсказки (не команды):

- The flag you want forces a rebuild and verifies output equality automatically. Search `nix build --help` for the option whose description mentions "rebuild ... and check that the result is the same."
  * Нужный флаг заставляет выполнить пересборку и автоматически проверяет равенство вывода. Найди в `nix build --help` опцию, в описании которой есть «rebuild ... and check that the result is the same».
- `readlink result` gives you the path. `sha256sum result` gives you the tarball hash for the Docker image.
  * `readlink result` даёт тебе путь. `sha256sum result` даёт хеш tar-архива для Docker-образа.
- If your two paths differ, your build has a non-determinism. Common causes: a `created = "now"` you forgot to fix, a `mkDerivation` with an unpinned `fetch*` input, or a missing `flake.lock`.
  * Если пути различаются, в сборке есть недетерминизм. Частые причины: забытый `created = "now"`, `mkDerivation` с незакреплённым входом `fetch*` или отсутствующий `flake.lock`.

**Paste into `labs/submission18.md`:**

**Вставь в `labs/submission18.md`:**

- The two commands you ran (the order matters — you'll figure out the right ones from `nix build --help`)
  * Две команды, которые ты выполнил (порядок важен — правильные ты поймёшь из `nix build --help`)
- The captured `readlink result` from both runs of `default` — same string
  * Зафиксированный `readlink result` из обоих запусков `default` — одна и та же строка
- The captured `sha256sum result` from two clean rebuilds of `dockerImg` — same digest
  * Зафиксированный `sha256sum result` из двух чистых пересборок `dockerImg` — один и тот же дайджест
- Nix's own confirmation line (something like `checking outputs of '/nix/store/<drv>'... IDENTICAL store path both builds`)
  * Собственную строку подтверждения Nix (что-то вроде `checking outputs of '/nix/store/<drv>'... IDENTICAL store path both builds`)
- A short paragraph answering: how does `flake.lock` prevent the "works on my machine" failure mode that `requirements.txt` (Lab 1) and a Helm image tag (Lab 10) cannot?
  * Короткий абзац с ответом: как `flake.lock` предотвращает сценарий «у меня локально работает», которого не могут предотвратить `requirements.txt` (лаба 1) и тег образа в Helm (лаба 10)?

> 🏆 **This is the whole point of the lab.** A flake that builds isn't enough. A flake that builds **the same store path twice** is what "reproducible" actually means.
>
> 🏆 **В этом и вся суть лабы.** Флейка, которая собирается, недостаточно. Флейка, которая собирается **в один и тот же путь в хранилище дважды**, — вот что на самом деле означает «воспроизводимо».

---

## How to Submit

Как сдать

```bash
git switch -c lab18
git add labs/lab18/flake.nix labs/lab18/flake.lock labs/lab18/app/
git add labs/submission18.md
git commit -m "feat(lab18): Nix reproducible build with flake + dockerTools"
git push -u origin lab18
```

Open **two** PRs:

Открой **два** PR:

- `your-fork:lab18` → `course-repo:master` *(reviewed)*
  * `your-fork:lab18` → `course-repo:master` *(проверяется)*
- `your-fork:lab18` → `your-fork:master` *(merges into your own main when done)*
  * `your-fork:lab18` → `your-fork:master` *(вливается в твой собственный main, когда закончишь)*

PR checklist:

Чек-лист PR:

```text
- [ ] Task 1 — flake.nix written by hand, flake.lock committed, app builds
- [ ] Task 2 — dockerImg output added; image loads and runs; comparison to Lab 2 Dockerfile documented
- [ ] Bonus — two independent builds land on the identical store path; sha256sum equality proven for the image
```

Include in the PR description: **the store path** you reproduced (`/nix/store/<hash>-<name>`) and the **tarball sha256** from two `dockerImg` builds. Those two strings are the artifact of this lab.

В описание PR включи: **путь в хранилище**, который ты воспроизвёл (`/nix/store/<hash>-<name>`), и **sha256 tar-архива** из двух сборок `dockerImg`. Эти две строки — артефакт этой лабы.

---

## Acceptance Criteria

Критерии приёмки

### Task 1 — Flake builds the app (6 pts)

Задача 1 — флейк собирает приложение (6 баллов)

- ✅ `labs/lab18/flake.nix` written from the skeleton (`description`, `inputs`, `outputs` all filled)
  * ✅ `labs/lab18/flake.nix` написан по скелету (заполнены `description`, `inputs`, `outputs`)
- ✅ `inputs.nixpkgs.url` pinned to a **release tag** (e.g. `nixos-25.11`) — **not** `nixpkgs-unstable`
  * ✅ `inputs.nixpkgs.url` закреплён за **релизным тегом** (например, `nixos-25.11`) — **не** `nixpkgs-unstable`
- ✅ `packages.${system}.default` builds with one of `writeShellApplication`, `mkDerivation`, `buildPythonApplication`, or `buildGoModule` — choice defended in `submission18.md`
  * ✅ `packages.${system}.default` собирается одним из билдеров `writeShellApplication`, `mkDerivation`, `buildPythonApplication` или `buildGoModule` — выбор обоснован в `submission18.md`
- ✅ `flake.lock` present, **committed**, and shown in the submission with the `locked.rev` field highlighted
  * ✅ `flake.lock` присутствует, **закоммичен** и показан в отчёте с подсвеченным полем `locked.rev`
- ✅ `nix build` produces a working binary at `result/bin/<name>` that runs the Lab 1 service logic
  * ✅ `nix build` даёт работающий бинарник `result/bin/<name>`, выполняющий логику сервиса из лабы 1

### Task 2 — `dockerTools` image (4 pts)

Задача 2 — образ `dockerTools` (4 балла)

- ✅ `packages.${system}.dockerImg` added to the flake
  * ✅ `packages.${system}.dockerImg` добавлен во флейк
- ✅ `buildLayeredImage` `contents` includes the Task-1 package
  * ✅ `contents` у `buildLayeredImage` включает пакет из задачи 1
- ✅ `config.Cmd` points at the binary in exec form
  * ✅ `config.Cmd` указывает на бинарник в exec-форме
- ✅ `config.ExposedPorts` set
  * ✅ `config.ExposedPorts` задан
- ✅ `created` is a **fixed epoch string** (not `"now"`) — typically `"1970-01-01T00:00:01Z"`
  * ✅ `created` — **строка с фиксированной эпохой** (не `"now"`), обычно `"1970-01-01T00:00:01Z"`
- ✅ `nix build .#dockerImg` produces a tarball; `docker load < result` succeeds; the loaded image runs and serves `/health`
  * ✅ `nix build .#dockerImg` создаёт tar-архив; `docker load < result` проходит успешно; загруженный образ работает и отдаёт `/health`
- ✅ Two `docker build` runs of the Lab 2 Dockerfile show **differing** `docker save | sha256sum` outputs, captured in `submission18.md`
  * ✅ Два запуска `docker build` Dockerfile из лабы 2 дают **различающиеся** выводы `docker save | sha256sum`, зафиксированные в `submission18.md`

### Bonus — Bit-for-bit proof (2 pts)

Бонус — побитовое доказательство (2 балла)

- ✅ Two independent `nix build .#default` runs (the second forcing a real rebuild) produce the **identical** `/nix/store/<hash>-<name>` path
  * ✅ Два независимых запуска `nix build .#default` (второй принудительно пересобирает) дают **идентичный** путь `/nix/store/<hash>-<name>`
- ✅ Two independent `nix build .#dockerImg` runs produce the **identical** `sha256sum result` digest
  * ✅ Два независимых запуска `nix build .#dockerImg` дают **идентичный** дайджест `sha256sum result`
- ✅ Nix's own "identical store path / checking outputs" confirmation captured
  * ✅ Зафиксировано собственное подтверждение Nix «identical store path / checking outputs»
- ✅ Paragraph in `submission18.md` explaining the role of `flake.lock` vs `requirements.txt` / Helm image tag
  * ✅ Абзац в `submission18.md` о роли `flake.lock` в сравнении с `requirements.txt` / тегом образа в Helm

---

## Rubric

Рубрика

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — Flake + lock + build | **6** | Outer shape filled, nixpkgs pinned to a release tag, derivation builder chosen correctly, `flake.lock` committed, binary runs |
| **Task 2** — `dockerTools` image | **4** | Image output added, `created` epoch trick applied, image loads & runs, contrast vs Lab 2 documented with both `sha256sum`s |
| **Bonus** — Bit-for-bit proof | **2** | Two builds → identical store path AND identical image sha256; Nix's own equality check captured |
| **Total** | **12** | 10 main + 2 bonus |

| Задача | Баллы | Критерии |
|------|-------:|----------|
| **Задача 1** — флейк + lock + сборка | **6** | Внешняя структура заполнена, nixpkgs закреплён за релизным тегом, билдер derivation выбран верно, `flake.lock` закоммичен, бинарник запускается |
| **Задача 2** — образ `dockerTools` | **4** | Вывод образа добавлен, применён трюк с `created`, образ загружается и работает, сравнение с лабой 2 задокументировано с обеими суммами `sha256sum` |
| **Бонус** — побитовое доказательство | **2** | Две сборки → идентичный путь в хранилище И идентичный sha256 образа; зафиксирована собственная проверка равенства от Nix |
| **Итого** | **12** | 10 основных + 2 бонусных |

**Grading guidance:**

**Рекомендации по оцениванию:**

- **10/10 main** — Flake builds, image loads, proof of reproducibility is clean, `flake.lock` discussion shows understanding of *why* it matters
  * **10/10 основных** — флейк собирается, образ загружается, доказательство воспроизводимости чистое, обсуждение `flake.lock` показывает понимание, *почему* он важен
- **8–9/10** — Builds work; reproducibility claim shown but missing one of the headline artifacts (store path **or** image sha256)
  * **8–9/10** — сборки работают; воспроизводимость показана, но отсутствует один из главных артефактов (путь в хранилище **или** sha256 образа)
- **6–7/10** — Task 1 works; Task 2 image builds but the `created` epoch isn't applied, or the Lab-2 contrast isn't measured
  * **6–7/10** — задача 1 работает; образ из задачи 2 собирается, но эпоха `created` не применена или сравнение с лабой 2 не измерено
- **<6/10** — Flake doesn't build, `flake.lock` missing, or `nixpkgs-unstable` used (reproducibility not actually achieved)
  * **<6/10** — флейк не собирается, `flake.lock` отсутствует или использован `nixpkgs-unstable` (воспроизводимость фактически не достигнута)

---

## Resources

Ресурсы

<details>
<summary>📚 Nix fundamentals</summary>

📚 Основы Nix

- [nix.dev](https://nix.dev/) — official tutorials
  * [nix.dev](https://nix.dev/) — официальные руководства
- [Zero to Nix](https://zero-to-nix.com/) — Determinate Systems' beginner track (flakes-first)
  * [Zero to Nix](https://zero-to-nix.com/) — вводный курс от Determinate Systems (сначала флейки)
- [Nix Pills](https://nixos.org/guides/nix-pills/) — deep dive for the curious
  * [Nix Pills](https://nixos.org/guides/nix-pills/) — глубокое погружение для любопытных
- [NixOS package search](https://search.nixos.org/) — find the `pkgs.<name>` you need
  * [Поиск пакетов NixOS](https://search.nixos.org/) — найти нужный тебе `pkgs.<name>`

</details>

<details>
<summary>📦 Flakes, dockerTools, and proof</summary>

📦 Флейки, dockerTools и доказательство

- [Flakes — NixOS Wiki](https://wiki.nixos.org/wiki/Flakes)
  * [Flakes — NixOS Wiki](https://wiki.nixos.org/wiki/Flakes)
- [Practical Nix Flakes — Serokell](https://serokell.io/blog/practical-nix-flakes)
  * [Practical Nix Flakes — Serokell](https://serokell.io/blog/practical-nix-flakes)
- [FlakeHub](https://flakehub.com/) — SemVer flakes registry
  * [FlakeHub](https://flakehub.com/) — реестр флейков с SemVer
- [`dockerTools` reference](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools)
  * [Справочник по `dockerTools`](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-dockerTools)
- [`nix build --help`](https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-build.html) — the flag you want for the Bonus lives here
  * [`nix build --help`](https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-build.html) — нужный для бонуса флаг живёт здесь
- [Reproducible Builds project](https://reproducible-builds.org/) — why this matters beyond Nix
  * [Проект Reproducible Builds](https://reproducible-builds.org/) — почему это важно вне Nix

</details>

<details>
<summary>🐧 Container-Nix vs host-Nix</summary>

🐧 Nix в контейнере vs Nix на хосте

- [`nixos/nix` Docker images](https://hub.docker.com/r/nixos/nix) — pinned tags (use `2.34.0` or any 2.25+)
  * [Docker-образы `nixos/nix`](https://hub.docker.com/r/nixos/nix) — закреплённые теги (используй `2.34.0` или любой 2.25+)
- [Determinate Nix](https://determinate.systems/) — host installer with flakes-on-default
  * [Determinate Nix](https://determinate.systems/) — установщик на хост с флейками по умолчанию
- [Lix](https://lix.systems/) — community fork, drop-in `nix` replacement
  * [Lix](https://lix.systems/) — форк от сообщества, drop-in замена `nix`

</details>

<details>
<summary>⚠️ Common Pitfalls (from real dry-runs)</summary>

⚠️ Частые ошибки (из реальных прогонов)

- **Flakes only see git-tracked files** — if you `nix build` before `git add flake.nix flake.lock app/`, Nix reports `path 'flake.nix' does not exist` or evaluates against an empty source tree. **Always `git add` first**, then build. This is THE most common Lab-18 mistake. (Reference dry-run hit it.)
  * **Флейки видят только файлы под контролем git** — если запустить `nix build` до `git add flake.nix flake.lock app/`, Nix сообщит `path 'flake.nix' does not exist` или станет вычислять пустое дерево исходников. **Сначала всегда `git add`**, потом сборка. Это САМАЯ частая ошибка в лабе 18. (Эталонный прогон на неё наткнулся.)
- **`safe.directory` inside the `nixos/nix` container** — the container runs as `root` (uid 0); your bind-mounted repo on the host is owned by your uid. `nix build` invokes libgit2, which refuses with `repository path '/work/' is not owned by current user`. Fix once: `git config --global --add safe.directory /work` inside the container. This is a **containerized-Nix artifact**, not a flake defect; host-Nix doesn't hit it.
  * **`safe.directory` внутри контейнера `nixos/nix`** — контейнер работает от `root` (uid 0), а твой репозиторий, смонтированный с хоста, принадлежит твоему uid. `nix build` вызывает libgit2, который отказывает с `repository path '/work/' is not owned by current user`. Исправляется один раз: `git config --global --add safe.directory /work` внутри контейнера. Это **артефакт контейнерного Nix**, а не дефект флейка; на хосте такой проблемы нет.
- **`dockerTools.buildLayeredImage` with `created = "now"` defeats reproducibility** — every build will produce a different image hash. Reading the docs you'll see `"now"` mentioned as the default for `buildImage`; that's exactly what you must override. Use `"1970-01-01T00:00:01Z"` (one second past the epoch — the literal epoch is rejected by some tooling).
  * **`dockerTools.buildLayeredImage` с `created = "now"` убивает воспроизводимость** — каждая сборка даст другой хеш образа. В документации ты увидишь `"now"` как значение по умолчанию для `buildImage`; именно его и нужно переопределить. Используй `"1970-01-01T00:00:01Z"` (одна секунда после эпохи — буквальную эпоху некоторые инструменты отвергают).
- **`inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable"`** — the branch tip moves daily, so today's build won't match next week's. The whole point of reproducibility is gone. Use a release tag (`nixos-25.11`) or pin a specific commit SHA.
  * **`inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixpkgs-unstable"`** — вершина ветки движется каждый день, поэтому сегодняшняя сборка не совпадёт с прошлой неделей. Весь смысл воспроизводимости исчезает. Используй релизный тег (`nixos-25.11`) или закрепи конкретный SHA коммита.
- **`nix flake update` rewrites `flake.lock`** — don't run it casually before a proof of reproducibility, or you'll change the closure under your own feet. Run it once after writing `flake.nix`, commit `flake.lock`, then leave it alone for the rest of the lab.
  * **`nix flake update` перезаписывает `flake.lock`** — не запускай его небрежно перед доказательством воспроизводимости, иначе изменишь замыкание у себя под ногами. Запусти один раз после написания `flake.nix`, закоммить `flake.lock` и больше не трогай до конца лабы.
- **`writeShellApplication` rejects unset variables** — it injects `set -euo pipefail` for you. If your script reads `$HOSTNAME` and you forgot to add `runtimeInputs = [ pkgs.inetutils ];` (or set the variable explicitly), the build passes but the script fails at runtime. Cleaner to use a Nix-resolved value.
  * **`writeShellApplication` отвергает незаданные переменные** — он сам добавляет `set -euo pipefail`. Если скрипт читает `$HOSTNAME`, а ты забыл добавить `runtimeInputs = [ pkgs.inetutils ];` (или явно задать переменную), сборка пройдёт, но скрипт упадёт при запуске. Чище использовать значение, разрешённое через Nix.
- **`nix-store --delete` to "force a rebuild" is brittle** — Nix may legitimately recover the cached output from elsewhere. Use `nix build --rebuild` for the Bonus proof; it explicitly does what you want (real rebuild + output-equality check).
  * **`nix-store --delete` для «принудительной пересборки» ненадёжен** — Nix может законно восстановить закешированный вывод откуда-то ещё. Для бонусного доказательства используй `nix build --rebuild`: он явно делает то, что нужно (настоящая пересборка + проверка равенства вывода).
- **`format = "other"` in `buildPythonApplication`** — required when your source has no `pyproject.toml` / `setup.py`. Without it, Nix tries to invoke `pip install .` and fails. Single-file Flask apps almost always need `format = "other"`.
  * **`format = "other"` в `buildPythonApplication`** — обязателен, если в исходниках нет `pyproject.toml` / `setup.py`. Без него Nix пытается вызвать `pip install .` и падает. Одностраничным Flask-приложениям почти всегда нужен `format = "other"`.
- **`USER` in `dockerTools` config is a string, not a number** — `config.User = "1000:1000"` works; `config.User = 1000` evaluates fine but Docker may complain on `docker run`. Use the `"uid:gid"` form.
  * **`USER` в конфиге `dockerTools` — строка, а не число** — `config.User = "1000:1000"` работает; `config.User = 1000` вычисляется нормально, но Docker может пожаловаться при `docker run`. Используй форму `"uid:gid"`.

</details>

<details>
<summary>💡 Tips</summary>

💡 Советы

1. Store paths are content-addressed: same inputs → same output hash. Always.
   * Пути в хранилище адресуются по содержимому: одинаковые входы → одинаковый хеш вывода. Всегда.
2. `nix-shell -p <pkg>` (classic CLI) or `nix shell nixpkgs#<pkg>` (new CLI) drops you into a shell with that package on `$PATH` — great for testing a dep before adding it to your derivation.
   * `nix-shell -p <pkg>` (классический CLI) или `nix shell nixpkgs#<pkg>` (новый CLI) погружает тебя в шелл с этим пакетом в `$PATH` — удобно проверить зависимость до добавления в derivation.
3. `nix-collect-garbage -d` frees disk space taken by old builds.
   * `nix-collect-garbage -d` освобождает место на диске, занятое старыми сборками.
4. Builds are sandboxed with **no network** — every dependency must be declared. If the build mysteriously needs internet, you're missing an input.
   * Сборки выполняются в песочнице **без сети** — каждая зависимость должна быть объявлена. Если сборке вдруг понадобился интернет, ты упустил какой-то вход.
5. Use a **release branch** for `nixpkgs` (`nixos-25.11`, `nixos-24.05`) — *not* unstable. The release tag is what makes "build today and in 2036" make sense.
   * Используй для `nixpkgs` **релизную ветку** (`nixos-25.11`, `nixos-24.05`) — *не* unstable. Именно релизный тег делает осмысленной фразу «собрать сегодня и в 2036 году».
6. `nix flake show` lists every output your flake exposes. Useful when your `packages.${system}` block grows.
   * `nix flake show` перечисляет все выходы, которые предоставляет твой флейк. Полезно, когда блок `packages.${system}` разрастается.

</details>

---

## Looking Ahead

Что дальше

Nix is **orthogonal** to the orchestration you spent the rest of the course mastering — it solves reproducibility, not scheduling. Where you'll meet it again:

Nix **ортогонален** оркестрации, которую ты осваивал весь остаток курса, — он решает задачу воспроизводимости, а не планирования. Где ты снова с ним встретишься:

| Domain | What Nix adds |
|---|---|
| CI | One-line `nix build` instead of "install N tools" pre-steps; build cache via Cachix |
| Dev shells | `nix develop` gives every contributor identical Python/Node/Go versions — no `venv`/`asdf`/`nvm` drift |
| Image builds | `dockerTools` images plug into the same registries / Helm charts you used in Labs 10-13 |
| Security audit | The closure is the artifact; SBOM is by construction |
| NixOS | If you ever want the same purity at the **OS** level |

| Область | Что добавляет Nix |
|---|---|
| CI | Однострочный `nix build` вместо предварительных шагов «установи N инструментов»; кеш сборок через Cachix |
| Dev-шеллы | `nix develop` даёт всем участникам одинаковые версии Python/Node/Go — без расхождений из-за `venv`/`asdf`/`nvm` |
| Сборка образов | Образы `dockerTools` подключаются к тем же реестрам / Helm-чартам, что ты использовал в лабах 10–13 |
| Аудит безопасности | Замыкание зависимостей и есть артефакт; SBOM получается по построению |
| NixOS | Если когда-нибудь захочешь такой же чистоты на уровне **ОС** |

---

**Hot take:** containers solved *deployment*. Nix solves *the build*. Two different problems, two different tools — and the only way to know yours actually ships what you wrote is to build it twice and watch the hashes match. ❄️

**Резко, но честно:** контейнеры решили задачу *деплоя*. Nix решает задачу *сборки*. Две разные проблемы, два разных инструмента — и единственный способ убедиться, что твоя сборка действительно отгружает написанное тобой, — собрать дважды и увидеть, что хеши совпали. ❄️
