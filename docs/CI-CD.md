# CI/CD (Swarm + SOPS)

Цей документ описує end-to-end CI/CD flow для стеків, що деплояться через `shared-ci-cd.yml` у режимі Swarm + SOPS.

## 1) Архітектура flow

1. Per-repo `.github/workflows/main.yml` викликає dispatcher:
   - `mzhk-repo/shared-workflows/.github/workflows/shared-ci-cd.yml@main`
2. Dispatcher маршрутизує:
   - `use_ansible: false` -> `shared-ci-cd-compose.yml`
   - `use_ansible: true` -> `shared-ci-cd-swarm.yml`
3. У Swarm flow:
   - вибір `env.dev.enc` або `env.prod.enc`
   - decrypt через `SOPS_AGE_KEY`
   - pre-flight recipient check
   - remote запуск per-repo orchestration script
   - `ansible-playbook --tags secrets` перед `docker stack deploy`
   - cleanup plaintext/ephemeral ключів

## 2) GitHub Environment Secrets (UI setup)

Налаштувати окремо для Environment `development` і `production`.

Обов'язкові secrets:

| Secret | Призначення |
|---|---|
| `SOPS_AGE_KEY` | Private age key (`AGE-SECRET-KEY-...`) для decrypt `env.*.enc` |
| `SERVER_HOST` | Хост Swarm manager для конкретного Environment |
| `SERVER_USER` | SSH user (зазвичай `ansible_usr`) |
| `SERVER_SSH_KEY` | Private SSH key для `SERVER_USER` |
| `SSH_HOST_KEY_PUB` | Public host key сервера |
| `SERVER_SSH_PORT` | Необов'язково, якщо SSH не на `22` |
| `DEPLOY_PROJECT_DIR` | Шлях repo на remote host |
| `INFRA_REPO_PAT` | PAT для доступу до infra/Ansible repo (якщо використовується у flow) |

Org-level/shared secrets (за потреби):

| Secret | Призначення |
|---|---|
| `TS_OAUTH_CLIENT_ID` | Tailscale OAuth client id |
| `TS_OAUTH_SECRET` | Tailscale OAuth secret |

Рекомендації:
- Для `development` і `production` можуть бути різні `SERVER_HOST`, `DEPLOY_PROJECT_DIR`, `SERVER_SSH_PORT`.
- `SOPS_AGE_KEY` може бути той самий для обох environments.
- Не зберігати plaintext `.env` у репозиторіях.
- Покрокове заведення нового SSH deploy-користувача описано в `docs/SSH-DEPLOY-USER-RUNBOOK.md`.

## 3) Per-repo `main.yml` (приклад)

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
  pull-requests: read

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
      docker_image_name: your-repo-name
      orchestration_script_path: 'scripts/deploy-orchestrator-swarm.sh'
      infra_repo_path: '/opt/Ansible'
    secrets:
      TS_OAUTH_CLIENT_ID: ${{ secrets.TS_OAUTH_CLIENT_ID }}
      TS_OAUTH_SECRET: ${{ secrets.TS_OAUTH_SECRET }}
      SERVER_HOST: ${{ secrets.SERVER_HOST }}
      SERVER_USER: ${{ secrets.SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.SERVER_SSH_KEY }}
      SERVER_SSH_PORT: ${{ secrets.SERVER_SSH_PORT }}
      SSH_HOST_KEY_PUB: ${{ secrets.SSH_HOST_KEY_PUB }}
      DEPLOY_PROJECT_DIR: ${{ secrets.DEPLOY_PROJECT_DIR }}
      SOPS_AGE_KEY: ${{ secrets.SOPS_AGE_KEY }}

  deploy-prod:
    if: |
      (github.event_name == 'pull_request' && github.base_ref == 'main') ||
      (github.event_name == 'push' && github.ref == 'refs/heads/main') ||
      (github.event_name == 'release' && startsWith(github.event.release.tag_name, 'v'))
    uses: mzhk-repo/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: production
      deploy: ${{ github.event_name == 'release' && startsWith(github.event.release.tag_name, 'v') }}
      use_ansible: true
      build_and_push_docker: false
      docker_image_name: your-repo-name
      orchestration_script_path: 'scripts/deploy-orchestrator-swarm.sh'
      infra_repo_path: '/opt/Ansible'
    secrets:
      TS_OAUTH_CLIENT_ID: ${{ secrets.TS_OAUTH_CLIENT_ID }}
      TS_OAUTH_SECRET: ${{ secrets.TS_OAUTH_SECRET }}
      SERVER_HOST: ${{ secrets.SERVER_HOST }}
      SERVER_USER: ${{ secrets.SERVER_USER }}
      SERVER_SSH_KEY: ${{ secrets.SERVER_SSH_KEY }}
      SERVER_SSH_PORT: ${{ secrets.SERVER_SSH_PORT }}
      SSH_HOST_KEY_PUB: ${{ secrets.SSH_HOST_KEY_PUB }}
      DEPLOY_PROJECT_DIR: ${{ secrets.DEPLOY_PROJECT_DIR }}
      SOPS_AGE_KEY: ${{ secrets.SOPS_AGE_KEY }}
```

## 4) Per-repo orchestration script template (Swarm)

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(cd "${SCRIPT_DIR}/.." && pwd)"

MODE="${ORCHESTRATOR_MODE:-noop}"
STACK_NAME="${STACK_NAME:-your-stack-name}"
ENV_FILE="${ORCHESTRATOR_ENV_FILE:-/tmp/env.decrypted}"

log() {
  printf '[deploy-orchestrator] %s\n' "$*"
}

detect_compose_file() {
  if [[ -f "docker-compose.yaml" ]]; then
    echo "docker-compose.yaml"
  elif [[ -f "docker-compose.yml" ]]; then
    echo "docker-compose.yml"
  else
    echo ""
  fi
}

run_ansible_secrets_if_configured() {
  local infra_repo_path environment inventory_env inventory_path playbook_path
  infra_repo_path="${INFRA_REPO_PATH:-}"
  environment="${ENVIRONMENT_NAME:-}"

  [[ -z "${infra_repo_path}" ]] && return 0
  [[ ! -d "${infra_repo_path}" ]] && { log "ERROR: INFRA_REPO_PATH not found"; exit 1; }
  command -v ansible-playbook >/dev/null 2>&1 || { log "ERROR: ansible-playbook not found"; exit 1; }

  case "${environment}" in
    development|dev) inventory_env="dev" ;;
    production|prod) inventory_env="prod" ;;
    *) log "ERROR: unsupported ENVIRONMENT_NAME=${environment}"; exit 1 ;;
  esac

  inventory_path="${infra_repo_path}/ansible/inventories/${inventory_env}/hosts.yml"
  playbook_path="${infra_repo_path}/ansible/playbooks/swarm.yml"
  [[ -f "${inventory_path}" && -f "${playbook_path}" ]] || { log "ERROR: ansible inventory/playbook missing"; exit 1; }

  ANSIBLE_CONFIG="${infra_repo_path}/ansible/ansible.cfg" \
    ansible-playbook -i "${inventory_path}" "${playbook_path}" --tags secrets
}

deploy_swarm() {
  local compose_file swarm_file raw_manifest deploy_manifest

  compose_file="$(detect_compose_file)"
  swarm_file="docker-compose.swarm.yml"
  raw_manifest="$(mktemp "${PROJECT_ROOT}/.${STACK_NAME}.stack.raw.XXXXXX.yml")"
  deploy_manifest="$(mktemp "${PROJECT_ROOT}/.${STACK_NAME}.stack.deploy.XXXXXX.yml")"
  trap 'rm -f "${raw_manifest:-}" "${deploy_manifest:-}"' RETURN

  [[ -n "${compose_file}" ]] || { log "ERROR: compose file not found"; exit 1; }
  [[ -f "${swarm_file}" ]] || { log "ERROR: docker-compose.swarm.yml missing"; exit 1; }

  if [[ ! -f "${ENV_FILE}" ]]; then
    [[ -f ".env" ]] && ENV_FILE=".env" || { log "ERROR: env file missing"; exit 1; }
  fi

  run_ansible_secrets_if_configured

  docker compose --env-file "${ENV_FILE}" -f "${compose_file}" -f "${swarm_file}" config > "${raw_manifest}"
  awk 'NR==1 && $1=="name:" {next} {print}' "${raw_manifest}" > "${deploy_manifest}"

  docker stack deploy -c "${deploy_manifest}" "${STACK_NAME}"
}

cd "${PROJECT_ROOT}"

case "${MODE}" in
  noop) log "No-op mode" ;;
  swarm) deploy_swarm ;;
  *) log "ERROR: unknown ORCHESTRATOR_MODE=${MODE}"; exit 1 ;;
esac
```

## 5) Як вручну запустити деплой через CI

Варіанти запуску:
- `dev`: зробити `push` у `dev` (або PR у `dev` для перевірок без deploy).
- `prod`: створити release tag `v*` на `main`.
- Повторний запуск: `Actions` -> потрібний workflow run -> `Re-run jobs`.

## 6) Troubleshooting

### 6.1 `error loading config ... cannot unmarshal !!seq into string`
Причина: `.sops.yaml` має `creation_rules[].age` як список, а не string.

Що робити:
- привести `creation_rules[].age` до string-формату;
- повторно зашифрувати/перевірити `env.*.enc`.

### 6.2 `no identity matched any of the recipients`
Причина: `SOPS_AGE_KEY` не відповідає recipient у `env.dev.enc`/`env.prod.enc`.

Що робити:
- перевірити ключ в Environment Secrets;
- перевірити recipient metadata у `env.*.enc`;
- перевипустити encrypted env правильним ключем.

### 6.3 `ssh: connect to host ... port 22: Connection refused`
Причина: SSH на не-22 порту.

Що робити:
- задати `SERVER_SSH_PORT` в Environment Secrets;
- оновити `SSH_HOST_KEY_PUB` для цього порту (`ssh-keyscan -p <port>`).

### 6.4 `service ... secret not found`
Причина: не створені/неоновлені Docker Secrets перед deploy.

Що робити:
- переконатися, що в orchestration викликається `ansible-playbook --tags secrets`;
- перевірити назви secret у `docker-compose.swarm.yml` та payload mapping в Ansible.

### 6.5 `MONITORING_NETWORK_NAME is not set`
Причина: змінна відсутня в decrypted env або shell env.

Що робити:
- додати `MONITORING_NETWORK_NAME` у `env.dev.enc`/`env.prod.enc`;
- переконатися, що `ORCHESTRATOR_ENV_FILE` коректний (`/tmp/env.decrypted`).

### 6.6 `network ... already exists`
Причина: конфлікт типу мережі/імені (bridge vs overlay).

Що робити:
- використовувати external overlay network pattern у swarm;
- перед deploy перевіряти `driver/scope` мережі;
- для несумісної мережі — змінити env-ім'я або прибрати стару мережу.

### 6.7 `services.<svc>.ports.<n>.published must be a integer`
Причина: у рендері `published` став рядком.

Що робити:
- додати нормалізацію `published: "8080"` -> `published: 8080` перед `docker stack deploy`.

### 6.8 `Permission denied` на тимчасовому manifest (`/tmp/*.stack.deploy.yml`)
Причина: повторне використання статичних шляхів з іншим owner.

Що робити:
- використовувати `mktemp` + `trap cleanup` у скрипті.

## 7) Security best practices

- `SOPS_AGE_KEY` зберігається **тільки** в GitHub Environment Secrets.
- Plaintext env (`/tmp/env.decrypted`) існує тільки в runtime job і має бути видалений у cleanup (`always()`).
- Не передавати секрети як CLI-аргументи; використовувати env/file-based підхід.
- Для Ansible/remote path використовувати ephemeral key-file + cleanup.
- У логах не друкувати значення secret-змінних.

## 8) DoD для CI/CD документації

- [x] Описано GitHub Environment Secrets setup (UI)
- [x] Додано приклади per-repo `main.yml`
- [x] Додано template `deploy-orchestrator-swarm.sh`
- [x] Додано troubleshooting для типових інцидентів
- [x] Додано security recommendations
