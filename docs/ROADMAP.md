## Фаза 8 — CI/CD Integration (Shared GitHub Actions Workflow)

**Пріоритет: P1**

### Мета
Автоматизувати SOPS-based deploy Swarm-стеків через GitHub Actions з дешифруванням `env.dev.enc` / `env.prod.enc` у CI runtime, із скупленням Docker Secrets через Ansible та розгортанням стеків через per-repo orchestration скрипти, з мінімізацією витоку секретів і збереженням source-of-truth в runbooks.

Репозиторії де потрібне впровадження:
1. `/opt/cloudflare-tunnel/`
2. `/opt/Traefik/`
3. `/opt/kdv-integrator/kdv-integrator-event/`
4. `/opt/Dspace/DSpace-docker/`
5. `/opt/Koha/koha-deploy/`
6. `/opt/Matomo-analytics/`
7. `/opt/victoriametrics-grafana/`

### Архітектурні принципи

1. **Ранбуки = source of truth для manual ops**: `/opt/Ansible/docs/RUNBOOKS/0X-*.md` описують покроково, як розгорнути стек на `dev-manager-01` вручну. CI автоматизує те ж.

2. **Спільний SOPS age key**: один private age key для всіх (dev+prod), зберігається **тільки** у GitHub Environment Secrets, ніколи на сервері.

3. **GitHub Environment Secrets для dev/prod розділення**: 
   - Environment "development" → `SOPS_AGE_KEY`, `SERVER_HOST` (dev), `SSH_*`, etc.
   - Environment "production" → `SOPS_AGE_KEY` (той же ключ), `SERVER_HOST` (prod), `SSH_*`, etc.
   - Кожна job має `environment: ${{ inputs.environment_name }}` → автоматично гранулює секрети

4. **Per-repo orchestration скрипти розділені за режимом деплою**:
   - `scripts/deploy-orchestrator.sh` = legacy compose path.
   - Якщо `scripts/deploy-orchestrator.sh` вже викликає `docker compose`, **його не чіпаємо**.
   - Для Swarm/SOPS додаємо окремий `scripts/deploy-orchestrator-swarm.sh`.
   - Swarm-скрипт відповідає за `ansible --tags secrets`, `docker stack deploy`, smoke-check і rollback logic.
   - **App-specific логіка деплою має бути у per-repo скриптах, не у shared workflow.**

5. **Shared workflows = оркестратори, не виконавці**:
   - `shared-ci-cd.yml` = dispatcher (backward-compatible entrypoint для всіх repo).
   - `shared-ci-cd-compose.yml` = legacy Docker Compose flow.
   - `shared-ci-cd-swarm.yml` = Docker Swarm + SOPS+age flow.

### Що робимо

#### 8.1 GitHub Environment Secrets (налаштувати вручну в GitHub UI)

Для **кожного** Environment (development + production):

| Secret | Значення | Примітка |
|--------|----------|---------|
| `SOPS_AGE_KEY` | Private age key (вихідний текст `AGE-SECRET-KEY-...`) | Один ключ для обох env; маскується у логах |
| `SERVER_HOST` | IP або hostname Swarm manager (dev: dev-manager-01, prod: prod-manager) | Environment-scoped |
| `SERVER_USER` | `ansible_usr` | Environment-scoped |
| `SERVER_SSH_KEY` | Private SSH key (ed25519) для `ansible_usr` | Environment-scoped |
| `SSH_HOST_KEY_PUB` | Public host key сервера (вихід `ssh-keyscan`) | Environment-scoped |
| `DEPLOY_PROJECT_DIR` | Path на сервері (наприклад, `/opt/cloudflare-tunnel`) | Environment-scoped |
| `INFRA_REPO_PAT` | GitHub PAT для клонування `/opt/shared-workflows` | Organization secret, або Environment |
| `TS_OAUTH_CLIENT_ID` | Tailscale OAuth ID | Organization level (shared) |
| `TS_OAUTH_SECRET` | Tailscale OAuth secret | Organization level (shared) |

**Важливо:** GitHub Environment Secrets автоматично **не видим у логах** і зберігаються окремо по Environment. Одне ім'я може мати різні значення у dev vs prod.

#### 8.2 Shared Workflow Enhancement (split на `compose` + `swarm`)

**Цільова структура shared-workflows:**

- `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml` — dispatcher (вхідна точка для backward compatibility).
- `/opt/shared-workflows/.github/workflows/shared-ci-cd-compose.yml` — legacy Docker Compose path.
- `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` — Swarm + SOPS+age path.

**Принцип маршрутизації:**
- `use_ansible: false` → викликається `shared-ci-cd-compose.yml`.
- `use_ansible: true` → викликається `shared-ci-cd-swarm.yml`.

```yaml
jobs:
  legacy-compose:
    if: inputs.use_ansible == false
    uses: ./.github/workflows/shared-ci-cd-compose.yml
  swarm-sops:
    if: inputs.use_ansible == true
    uses: ./.github/workflows/shared-ci-cd-swarm.yml
```

#### 8.3 Per-Repo Orchestration Scripts (compose + swarm)

**Правило сумісності для існуючих repo:**
- Якщо `scripts/deploy-orchestrator.sh` уже містить `docker compose` команди — **не модифікувати цей файл**.
- Для Swarm/SOPS створити окремий `scripts/deploy-orchestrator-swarm.sh`.

**Входи для swarm-скрипта (через CI):**
- `DEPLOY_PROJECT_DIR` — локальний шлях репо
- `ENVIRONMENT_NAME` — "development" або "production"
- `DEPLOY_REF` — git commit SHA
- `INFRA_REPO_PATH` — шлях до Ansible repo на manager

**Мінімальний приклад:**
- `scripts/deploy-orchestrator.sh` (legacy): залишаємо як є для `docker compose up -d`.
- `scripts/deploy-orchestrator-swarm.sh` (new): `ansible-playbook --tags secrets` + `docker stack deploy`.
- Якщо Swarm path тимчасово використовує `scripts/deploy-orchestrator.sh`, цей скрипт має обов'язково виконувати `ansible-playbook --tags secrets` **до** `docker stack deploy`.

#### 8.4 Per-Repo main.yml (оновлення)

**Вимога backward compatibility:** `main.yml` у кожному repo має лишатися сумісним з обома режимами (`compose` і `swarm`) через dispatcher `shared-ci-cd.yml`.

**Правило:**
- `uses` не змінюємо: `.../.github/workflows/shared-ci-cd.yml@main`.
- Перемикання режиму робимо тільки через `use_ansible` + `orchestration_script_path`.

```yaml
name: App Pipeline

on:
  push:
    branches: [dev, main]
    paths-ignore:
      - '**/*.md'
      - '.env.example'
      - '.gitignore'
  pull_request:
    branches: [dev, main]
    paths-ignore:
      - '**/*.md'
      - '.env.example'
      - '.gitignore'
  release:
    types: [published]

permissions:
  contents: read
  packages: write

jobs:
  deploy-dev:
    if: |
      (github.event_name == 'pull_request' && github.base_ref == 'dev') ||
      (github.event_name == 'push' && github.ref == 'refs/heads/dev')
    uses: mzhk-repo/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: development
      deploy: ${{ github.event_name == 'push' && github.ref == 'refs/heads/dev' }}
      use_ansible: true
      build_and_push_docker: false
      docker_image_name: app-name
      # Swarm path: окремий скрипт, legacy compose script не чіпаємо
      orchestration_script_path: 'scripts/deploy-orchestrator-swarm.sh'
    secrets:
      TS_OAUTH_CLIENT_ID: ${{ secrets.TS_OAUTH_CLIENT_ID }}
      TS_OAUTH_SECRET: ${{ secrets.TS_OAUTH_SECRET }}
      SERVER_HOST: ${{ secrets.SERVER_HOST }}
      SERVER_USER: ${{ secrets.SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.SERVER_SSH_KEY }}
      SSH_HOST_KEY_PUB: ${{ secrets.SSH_HOST_KEY_PUB }}
      DEPLOY_PROJECT_DIR: ${{ secrets.DEPLOY_PROJECT_DIR }}
      INFRA_REPO_PAT: ${{ secrets.INFRA_REPO_PAT }}
      SOPS_AGE_KEY: ${{ secrets.SOPS_AGE_KEY }}

  deploy-prod:
    if: |
      github.event_name == 'release' && startsWith(github.event.release.tag_name, 'v')
    uses: mzhk-repo/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: production
      deploy: true
      use_ansible: true
      build_and_push_docker: false
      docker_image_name: app-name
      orchestration_script_path: 'scripts/deploy-orchestrator-swarm.sh'
    secrets:
      TS_OAUTH_CLIENT_ID: ${{ secrets.TS_OAUTH_CLIENT_ID }}
      TS_OAUTH_SECRET: ${{ secrets.TS_OAUTH_SECRET }}
      SERVER_HOST: ${{ secrets.SERVER_HOST }}
      SERVER_USER: ${{ secrets.SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.SERVER_SSH_KEY }}
      SSH_HOST_KEY_PUB: ${{ secrets.SSH_HOST_KEY_PUB }}
      DEPLOY_PROJECT_DIR: ${{ secrets.DEPLOY_PROJECT_DIR }}
      INFRA_REPO_PAT: ${{ secrets.INFRA_REPO_PAT }}
      SOPS_AGE_KEY: ${{ secrets.SOPS_AGE_KEY }}
```

#### 8.5 Ansible Playbook (без змін, як є)

Playbook `playbooks/swarm.yml` вже підтримує `--tags secrets`:

```bash
# Фазу 7 уже підготувалась role + playbook
ansible-playbook -i inventories/dev \
                 playbooks/swarm.yml \
                 --tags secrets \
                 -e @/tmp/env.decrypted
```

Role `swarm_cluster` читає env vars (через `-e @file`) і створює Docker Secrets у Swarm.

#### 8.6 Документація: `docs/CI-CD.md` (нова)

Мета: описати end-to-end CI flow для користувачів/операторів.

**Структура `docs/CI-CD.md`:**
- GitHub Environment Secrets setup (manual UI steps)
- Per-repo main.yml примеры
- Orchestration script template
- Troubleshooting: "Why did deploy fail?", "How to manually trigger?", etc.
- Security best practices: "Where is SOPS_AGE_KEY stored?", "Can CI access plaintext?"

#### 8.7 Guardrails після реальних CI/CD інцидентів (обов'язково)

1. **SOPS config compatibility:** для `sops v3.10.2` у `.sops.yaml` поле `creation_rules[].age` має бути string (не YAML list), інакше decrypt падає на parse config.
2. **Recipient pre-flight check:** перед decrypt у `shared-ci-cd-swarm.yml` перевіряти, що public key з `SOPS_AGE_KEY` збігається з recipient у `env.dev.enc` / `env.prod.enc`.
3. **Remote key forwarding:** якщо orchestration викликає `ansible-playbook --tags secrets` на manager, `SOPS_AGE_KEY` має передаватись у remote execution context (ephemeral file + cleanup).
4. **SSH non-default port support:** shared workflows мають підтримувати `SERVER_SSH_PORT` і генерувати `known_hosts` у форматі `[host]:port` для порту, відмінного від `22`.
5. **Secrets-before-deploy order:** для Swarm path секрети завжди оновлюються (`ansible --tags secrets`) **до** `docker stack deploy`.

### Залежності
- Фази 0–7 завершені
- GitHub Environment Secrets налаштовані для dev + prod (SOPS_AGE_KEY, SERVER_HOST, SSH_*, DEPLOY_PROJECT_DIR)
- Ansible repo доступна через INFRA_REPO_PAT
- SOPS + age встановлено в CI runner (або install step у workflow)
- Per-repo `scripts/deploy-orchestrator.sh` (legacy compose) існує або не потребує змін
- Per-repo `scripts/deploy-orchestrator-swarm.sh` створений для Swarm/SOPS path

### Ризики
| Ризик | Мітигація |
|-------|-----------|
| `SOPS_AGE_KEY` у логах GitHub Actions | GitHub автоматично маскує Environment Secrets; додатково: `no_log: true` на sensitive steps, `--version` для утилітне (не передавати ключ як аргумент) |
| `.sops.yaml` несумісний із `sops 3.10.2` (`cannot unmarshal !!seq into string`) | Стандартизувати `creation_rules[].age` як string і перевіряти pre-commit/PR |
| Невідповідність recipient у `env.*.enc` і `SOPS_AGE_KEY` (`no identity matched`) | Recipient-match pre-flight у `shared-ci-cd-swarm.yml` до кроку decrypt |
| Plaintext env залишається на runner після job | `shred -vfz -n 3 /tmp/env.decrypted` перед exit (cleanup step з `always()`) |
| `ansible --tags secrets` на remote не бачить `SOPS_AGE_KEY` | Явно forward `SOPS_AGE_KEY` у remote orchestration (тимчасовий key-file + cleanup trap) |
| SSH недоступний на порті 22 | Використовувати `SERVER_SSH_PORT` і `[host]:port` у `known_hosts` |
| Diff dev vs prod deploy логіки | Orchestration скрипт отримує `ENVIRONMENT_NAME` і використовує той же код; все контролюється per-repo |
| Ansible repo недоступна або стара версія | `git fetch --all` у shared workflow перед use; перевіста INFRA_REPO_PAT |
| App repo не має `scripts/deploy-orchestrator-swarm.sh` | Fallback у swarm workflow: базовий `docker stack deploy` (але рекомендується мати окремий swarm-скрипт) |

### Definition of Done
- [x] Спільний SOPS age key передано у GitHub Environment Secrets (dev + prod)
- [x] Shared workflows розділено: `shared-ci-cd-compose.yml` + `shared-ci-cd-swarm.yml`, а `shared-ci-cd.yml` працює як dispatcher
- [x] `shared-ci-cd-swarm.yml` дешифрує env.dev.enc / env.prod.enc на runtime (in-memory)
- [x] `shared-ci-cd-swarm.yml` має pre-flight recipient check (`SOPS_AGE_KEY` ↔ `env.*.enc`) перед decrypt
- [x] `shared-ci-cd-swarm.yml` forward-ить `SOPS_AGE_KEY` у remote orchestration для `ansible --tags secrets`
- [x] Per-repo main.yml backward-compatible з обома режимами через `use_ansible` + `orchestration_script_path`
- [x] `scripts/deploy-orchestrator.sh` (compose) збережений без змін, якщо містить `docker compose`
- [x] `scripts/deploy-orchestrator-swarm.sh` присутній (або fallback у swarm workflow)
- [x] Для Swarm deploy secret refresh (`ansible --tags secrets`) виконується до `docker stack deploy`
- [x] Підтримано `SERVER_SSH_PORT` у shared workflows (ssh/scp + `[host]:port` known_hosts)
- [x] Перший push до dev branch тригерить deploy через CI (все успішно)
- [x] Перший release tag на main тригерить deploy до prod (все успішно)
- [x] Дешифрований env не залишається у логах / run artifacts
- [x] Cleanup step видаляє plaintext з runner перед exit
- [x] docs/CI-CD.md написано

### Чекліст для впровадження

**GitHub UI (manual one-time):**
- [x] Development Environment: SOPS_AGE_KEY, SERVER_HOST, SERVER_USER, SERVER_SSH_KEY, SSH_HOST_KEY_PUB, DEPLOY_PROJECT_DIR, INFRA_REPO_PAT
- [x] Development Environment: за потреби додати `SERVER_SSH_PORT` (якщо не 22)
- [x] Production Environment: SOPS_AGE_KEY (один ключ), SERVER_HOST (prod), SERVER_USER, SERVER_SSH_KEY, SSH_HOST_KEY_PUB, DEPLOY_PROJECT_DIR, INFRA_REPO_PAT
- [x] Production Environment: за потреби додати `SERVER_SSH_PORT` (якщо не 22)
- [x] Organization-level: TS_OAUTH_CLIENT_ID, TS_OAUTH_SECRET (якщо не Environment-specific)

**Shared Workflow repo (`mzhk-repo/shared-workflows`):**
- [x] Винести legacy flow у `.github/workflows/shared-ci-cd-compose.yml`
- [x] Винести Swarm/SOPS flow у `.github/workflows/shared-ci-cd-swarm.yml`
- [x] Оновити `.github/workflows/shared-ci-cd.yml` у dispatcher-mode
- [x] Перевірити routing `use_ansible=false/true`
- [x] Додати recipient-match pre-flight у swarm workflow перед decrypt
- [x] Додати forward `SOPS_AGE_KEY` на remote для orchestration/Ansible path
- [x] Додати support `SERVER_SSH_PORT` для `ssh/scp` + `[host]:port` у known_hosts

**Per-Repo (кожна з 7 stack):**
- [x] Оновити `.github/workflows/main.yml` з backward-compatible викликом dispatcher (`shared-ci-cd.yml`)
- [x] Перевірити, що `scripts/deploy-orchestrator.sh` (compose) не змінювався, якщо містить `docker compose`
- [x] Створити `scripts/deploy-orchestrator-swarm.sh` для Swarm/SOPS path
- [x] Перевірити `docker-compose.swarm.yml`: secrets/env_file/ labels налаштовані
- [x] Перевірити `env.dev.enc` + `env.prod.enc` присутні у repo
- [x] Перевірити `.pre-commit-config.yaml`: SOPS validation hook
- [x] Перевірити `.sops.yaml`: `creation_rules[].age` задано string-форматом (sops 3.10.2 compatible)
- [x] Тест: `docker compose --env-file env.dev.enc config` має успіти
- [x] Тест: push на dev branch → CI deploy до dev (очікується success)

**Documentation:**
- [x] Створити `docs/CI-CD.md` у /opt/shared-workflows/docs
- [x] Додати примеры per-repo main.yml и orchestration script
- [x] Описати Environment Secrets UI setup
- [x] Описати troubleshooting (deploy stuck, secret missing, etc.)
- [x] Оновити ROADMAP.md розділ "Структура docs/" — додати посилання на CI-CD.md

### Артефакти

**Shared Workflow:**
- `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml` (dispatcher)
- `/opt/shared-workflows/.github/workflows/shared-ci-cd-compose.yml` (new/updated)
- `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` (new/updated)

**Per-Repo (кожна з 7):**
- `.github/workflows/main.yml` (updated: backward-compatible dispatcher call)
- `scripts/deploy-orchestrator.sh` (legacy compose, unchanged якщо вже `docker compose`)
- `scripts/deploy-orchestrator-swarm.sh` (new/updated)
- `env.dev.enc` + `env.prod.enc` (already exist from Phase 7)
- `docker-compose.swarm.yml` (existing, no changes needed)

**Ansible Repo:**
- `ansible/playbooks/swarm.yml` (no changes: already supports `--tags secrets`)
- `ansible/roles/swarm_cluster/tasks/secrets.yml` (no changes: already handles SOPS vars)
- `docs/CI-CD.md` (new)