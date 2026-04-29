# CHANGELOG 2026 VOL 01

Статус: `active`
Відкрито: `2026-04-19`
Контекст: продовження Фази 8.


# 2026-04-17 — ROADMAP Phase 8 rewritten: CI/CD Integration with SOPS decrypt in GitHub Actions + per-repo orchestration scripts

- **Context:** На основі детального аналізу requirement-ів для Фази 8 (voir Phase 8 CI/CD flow) потрібно було переписати ROADMAP Фаза 8, щоб явно описати архітектуру: спільний SOPS age key у GitHub Environment Secrets, дешифрування env.dev.enc/env.prod.enc в CI runtime в-memory, Ansible playbook --tags secrets для Docker Secrets, per-repo orchestration скрипти як місце для app-specific логіки, та ранбуки як source of truth для manual ops.
- **Change:** Повністю переписано розділ "## Фаза 8 — CI/CD Integration (Shared GitHub Actions Workflow)" у `docs/ROADMAP.md` з наступними деталями:
  - **Архітектурні принципи:** 5 пунктів про розподіл відповідально (runbooks + CI automation, SOPS key scoping, Environment Secrets для dev/prod розділення, per-repo scripts vs shared workflow orchestration)
  - **8.1 GitHub Environment Secrets:** явна таблиця з dev + prod Environment mappings (SOPS_AGE_KEY як спільний, SERVER_HOST як різний)
  - **8.2 Shared Workflow Enhancement:** new bash код для SOPS install + decrypt step, Ansible checkout, per-repo orchestration script invocation через SSH, cleanup step для rm /tmp/env.decrypted
  - **8.3 Per-Repo Orchestration Script:** детальний template bash-скрипт (git fetch/checkout, Ansible --tags secrets invocation, docker compose config + stack deploy, force service update, smoke-check, cleanup)
  - **8.4 Per-Repo main.yml:** YAML приклад з `use_ansible: true` і параметрами для dev/prod jobs
  - **8.5 Ansible Playbook:** примітка що `playbooks/swarm.yml --tags secrets` вже готова (no changes)
  - **8.6 Documentation: docs/CI-CD.md:** нова документація (須創 в Фазі 8)
  - **Залежності, ризики, DoD, чекліст:** узгоджено з узгодженими вимогами (БЕЗ apt update/upgrade, force redeploy як per-repo logic, no duplication runbooks-CI)
- **Verification:** Оновлений ROADMAP містить: явну diff від попередньої версії (более коротке описання SOPS流 у старій версії), детальні YAML/bash приклади для CI integraton, посилання на per-repo orchestration як місце для app-specific деплою-логіки, 5 архітектурних принципів, явні GitHub UI steps та per-repo поточання.
- **Docs:** Оновлено також таблицю у розділі "Структура docs/" для явного посилання на `CI-CD.md` (Фаза 8) із деталізованим описом flow.
- **Notes:** Це чисто планова/документаційна зміна без інфраструктурних модифікацій; фактична імплементація (Ansible playbook updates, shared-ci-cd.yml rewrite, per-repo script creation) залишається на рівні Фази 8 under DoD execution notes.

## 2026-04-18 — Phase 8 step 1 (`/opt/cloudflare-tunnel`): додано per-repo orchestration script і підключено його в app workflow без switch на `use_ansible`

- **Context:** Розпочато фактичне виконання Фази 8 строго інкрементально з репозиторію `/opt/cloudflare-tunnel/`; перший крок мав бути безпечним і не ламати поточний shared workflow, який ще працює у legacy-режимі для `use_ansible`.
- **Change:** У `/opt/cloudflare-tunnel/scripts/deploy-orchestrator.sh` додано новий оркестратор із режимами `noop` (дефолт) та `swarm` (`ORCHESTRATOR_MODE=swarm`), з рендером `docker compose ... config` + `docker stack deploy`, fallback для env-файлу (`/tmp/env.decrypted` -> `.env`) і автодефолтами `CF_TUNNEL_TOKEN_SECRET_NAME` за `ENVIRONMENT_NAME`; у `/opt/cloudflare-tunnel/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` додано явний `orchestration_script_path: 'scripts/deploy-orchestrator.sh'`.
- **Verification:** `bash -n /opt/cloudflare-tunnel/scripts/deploy-orchestrator.sh` проходить успішно; `git -C /opt/cloudflare-tunnel status --short` показує лише очікувані зміни (`M .github/workflows/main.yml`, `?? scripts/deploy-orchestrator.sh`); `use_ansible` залишено `false` в обох jobs, тобто поточний CI поведінково не переведено на ризиковий path.
- **Risks:** Поки `shared-ci-cd.yml` не оновлено до SOPS-runtime flow з roadmap (SOPS decrypt + cleanup + Ansible `--tags secrets`), увімкнення `use_ansible: true` у repo не можна робити без ризику зламу деплою.
- **Rollback:** Видалити `/opt/cloudflare-tunnel/scripts/deploy-orchestrator.sh`, прибрати `orchestration_script_path` з `/opt/cloudflare-tunnel/.github/workflows/main.yml`.

## 2026-04-19 — Phase 8 step 2 (`/opt/shared-workflows`): `shared-ci-cd.yml` переведено з legacy Vault/apt path на SOPS runtime deploy-path для `use_ansible=true`

- **Context:** Після кроку 1 (підготовка `cloudflare-tunnel` оркестратора) потрібно було закрити головний blocking-gap: у shared workflow `use_ansible=true` ще використовував legacy flow (`ANSIBLE_VAULT_PASSWORD` + `apt install ansible`) і не відповідав новій архітектурі Фази 8.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml` для `workflow_call` додано `inputs.infra_repo_path` (default `/opt/Ansible`) і `secrets.SOPS_AGE_KEY`; повністю замінено legacy-блок `use_ansible == true` на Phase 8 runtime flow: валідація `SOPS_AGE_KEY`, інсталяція `sops` + `age` (без `apt update/upgrade`), decrypt `env.dev.enc`/`env.prod.enc` у `/tmp/env.decrypted`, передача env на remote host через `scp`, запуск repo-specific оркестратора в `ORCHESTRATOR_MODE=swarm` (або fallback `docker stack deploy`), cleanup plaintext на remote і runner.
- **Verification:** YAML проходить парсинг (`YAML_OK` через `yaml.safe_load`); diff показує очікувану заміну legacy Ansible/Vault блоку на SOPS runtime блок; `git -C /opt/shared-workflows status --short` містить лише `M .github/workflows/shared-ci-cd.yml`.
- **Risks:** Новий path для `use_ansible=true` тепер вимагає наявності `SOPS_AGE_KEY` і `env.dev.enc`/`env.prod.enc` у app repo; якщо repo не має `scripts/deploy-orchestrator.sh` з swarm-логікою, спрацює fallback deploy; інтеграція `ansible-playbook --tags secrets` поки делегується на per-repo orchestration script і не виконується централізовано в shared workflow.
- **Rollback:** Відкотити `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml` до попередньої ревізії з legacy-блоком.

## 2026-04-19 — Phase 8 step 3 (`/opt/shared-workflows`): монолітний `shared-ci-cd.yml` розділено на два профільні reusable workflow

- **Context:** Після оновлення SOPS-path у кроці 2 файл `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml` виріс до ~472 рядків і містив дві різні стратегії деплою в одному місці; потрібно було фізично розділити legacy compose та swarm+sops логіку.
- **Change:** Додано два нові workflow-файли: `/opt/shared-workflows/.github/workflows/shared-ci-cd-compose.yml` (тільки legacy deploy через `docker compose`) і `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` (Swarm + SOPS+age path); поточний `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml` перетворено на тонкий dispatcher (117 рядків), який маршрутизує у відповідний файл за `inputs.use_ansible`, щоб не ламати існуючі виклики repo-ів (`uses: .../shared-ci-cd.yml@main`).
- **Verification:** `yaml.safe_load` успішно парсить усі три файли (`YAML_OK_ALL`); поточні розміри: dispatcher `117` рядків, compose `305` рядків, swarm `374` рядки; `git -C /opt/shared-workflows status --short` показує очікувано `M .github/workflows/shared-ci-cd.yml` + `?? .github/workflows/shared-ci-cd-compose.yml` + `?? .github/workflows/shared-ci-cd-swarm.yml`.
- **Risks:** CI-checks секція наразі дублюється у двох профільних workflow (compose/swarm); при майбутніх змінах перевірок потрібно оновлювати обидва файли синхронно.
- **Rollback:** Видалити `/opt/shared-workflows/.github/workflows/shared-ci-cd-compose.yml` і `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml`, повернути попередню версію `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml`.

## 2026-04-19 — Phase 8 roadmap alignment: зафіксовано split workflows + правило окремого swarm-оркестратора + backward compatibility для per-repo `main.yml`

- **Context:** Після фактичного split shared workflow на `shared-ci-cd-compose.yml` і `shared-ci-cd-swarm.yml` потрібно було синхронізувати сам `docs/ROADMAP.md`, щоб план і реалізація Фази 8 не розходились.
- **Change:** У `docs/ROADMAP.md` (розділ Фази 8) оновлено: 1) архітектурні принципи і секцію `8.2` під трифайлову модель (`shared-ci-cd.yml` dispatcher + `shared-ci-cd-compose.yml` + `shared-ci-cd-swarm.yml`); 2) секцію `8.3` з явним правилом: якщо `scripts/deploy-orchestrator.sh` вже викликає `docker compose`, його не змінювати, а для Swarm/SOPS створювати окремий `scripts/deploy-orchestrator-swarm.sh`; 3) секцію `8.4` і чеклісти/DoD/артефакти під вимогу backward compatibility per-repo `main.yml` через dispatcher (`use_ansible` + `orchestration_script_path`).
- **Verification:** Перевірено цілісний блок Фази 8 після редагування (`sed -n '734,1165p' docs/ROADMAP.md`): присутні нові назви workflow-файлів, правило незмінності compose-скрипта та вимога backward compatibility для `main.yml`; зміни суто документаційні, без інфраструктурного деплою.
- **Risks:** У roadmap-прикладі `main.yml` показано Swarm-варіант (`use_ansible: true`); для repo, що ще не мігрували на Swarm path, потрібно тимчасово тримати `use_ansible: false` і `orchestration_script_path: scripts/deploy-orchestrator.sh` до окремого інкрементного переходу.
- **Rollback:** Відкотити зміни в `docs/ROADMAP.md` до попередньої ревізії Фази 8.

## 2026-04-19 — Phase 8 step 4 (`/opt/cloudflare-tunnel`): `main.yml` переключено на `use_ansible: true` під split shared workflow

- **Context:** Після split shared workflow і синхронізації roadmap потрібно було виконати запланований крок 3 для `cloudflare-tunnel`: переключити app pipeline на Swarm/SOPS path і додати необхідні параметри виклику.
- **Change:** У `/opt/cloudflare-tunnel/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` встановлено `use_ansible: true`; додано `infra_repo_path: '/opt/Ansible'`; у секцію `secrets` додано `SOPS_AGE_KEY`; збережено `orchestration_script_path: 'scripts/deploy-orchestrator.sh'` для сумісності з наявним оркестратором repo.
- **Verification:** YAML синтаксис валідний (`YAML_OK`); `git diff` показує очікувані зміни лише в `main.yml`; `git -C /opt/cloudflare-tunnel status --short` показує `M .github/workflows/main.yml` і наявний раніше `?? scripts/deploy-orchestrator.sh`.
- **Risks:** Поточний `scripts/deploy-orchestrator.sh` у cloudflare-tunnel не викликає `ansible-playbook --tags secrets`; при відсутньому/простроченому Docker Secret `cf_tunnel_token_*` deploy може потребувати окремого кроку з оновленням secret або переходу на `scripts/deploy-orchestrator-swarm.sh`.
- **Rollback:** Повернути `use_ansible: false` у `deploy-dev`/`deploy-prod` і прибрати додані `infra_repo_path` + `SOPS_AGE_KEY` з `/opt/cloudflare-tunnel/.github/workflows/main.yml`.

## 2026-04-19 — Phase 8 hotfix (`/opt/cloudflare-tunnel`): CI decrypt fix для `sops 3.10.2` (формат `.sops.yaml`)

- **Context:** Після переключення `cloudflare-tunnel` на `use_ansible: true` deploy падав у GitHub Actions на кроці decrypt з помилкою `error loading config ... line 4: cannot unmarshal !!seq into string`.
- **Root Cause:** У `/opt/cloudflare-tunnel/.sops.yaml` поле `creation_rules[].age` було задане як YAML-список, тоді як `sops 3.10.2` (встановлюється у shared workflow) очікує string-формат.
- **Change:** У `/opt/cloudflare-tunnel/.sops.yaml` змінено `age` з list-формату на string-формат (один recipient в одному рядку).
- **Verification:** Локально відтворено падіння на `sops 3.10.2`; після фіксу decrypt успішний як на `sops 3.10.2` (`OK_3102`), так і на поточній локальній версії (`OK_LOCAL`).
- **Rollback:** Повернути попередню версію `/opt/cloudflare-tunnel/.sops.yaml` (list-формат `age`) — **не рекомендовано**, бо знову ламатиме CI на `sops 3.10.2`.

## 2026-04-19 — Phase 8 guardrail (`/opt/shared-workflows`): явна перевірка відповідності `SOPS_AGE_KEY` recipient перед decrypt

- **Context:** Після виправлення `.sops.yaml` у `cloudflare-tunnel` наступний інцидент у CI показав помилку `no identity matched any of the recipients`, що вказує на mismatch між GitHub secret `SOPS_AGE_KEY` і recipient в `env.*.enc`.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` посилено кроки `Validate required SOPS secret` + `Decrypt env file with SOPS`: додано перевірку, що `SOPS_AGE_KEY` не є public `age1...` і містить `AGE-SECRET-KEY-`; перед decrypt обчислюється public key із secret і звіряється з `sops_age__list_*__map_recipient` у `env.*.enc`; decrypt переведено на `SOPS_AGE_KEY_FILE` з тимчасовим key-file cleanup.
- **Verification:** `yaml.safe_load` для `shared-ci-cd-swarm.yml` проходить (`YAML_OK`); diff містить очікувані guard-перевірки і recipient-match check.
- **Risks:** Якщо encrypted file має нетиповий формат metadata без `sops_age__list_*__map_recipient`, попередня explicit-match перевірка може бути пропущена; у такому разі фінальна валідація лишається на самому `sops --decrypt`.
- **Rollback:** Відкотити зміни у `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` до попередньої ревізії без guardrail-перевірок.

## 2026-04-19 — Phase 8 hotfix (`/opt/shared-workflows` + `/opt/cloudflare-tunnel`): підтримка нестандартного SSH порту в CI deploy

- **Context:** На кроці `Deploy on remote host` у GitHub Actions виникало `ssh: connect to host *** port 22: Connection refused` / `scp: Connection closed`, що вказує на недоступність SSH на порту 22 (сервер очікує інший `ssh_port`).
- **Change:** У shared workflows додано опційний secret `SERVER_SSH_PORT` і прокидання його у dispatcher: `/opt/shared-workflows/.github/workflows/shared-ci-cd.yml`, `/opt/shared-workflows/.github/workflows/shared-ci-cd-compose.yml`, `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml`; `ssh/scp` переведено на `-p/-P ${SERVER_SSH_PORT:-22}`; генерацію `~/.ssh/known_hosts` оновлено під формат `[host]:port` для нестандартних портів. У `/opt/cloudflare-tunnel/.github/workflows/main.yml` додано мапінг `SERVER_SSH_PORT: ${{ secrets.SERVER_SSH_PORT }}` для `deploy-dev` і `deploy-prod`.
- **Verification:** YAML-парсинг усіх оновлених workflow проходить (`YAML_OK_ALL`); diff підтверджує додавання `SERVER_SSH_PORT` і використання порту в `ssh/scp` + `known_hosts`.
- **Risks:** Якщо `SSH_HOST_KEY_PUB` у GitHub Secrets збережений не для того порту/хоста (або не у форматі без префікса host), StrictHostKeyChecking може зупиняти конект; для нестандартного порту бажано перевзяти ключ через `ssh-keyscan -p <port> <host>`.
- **Rollback:** Прибрати `SERVER_SSH_PORT` з workflow і повернути hardcoded поведінку порту 22 (не рекомендовано для інфраструктури з custom ssh_port).

## 2026-04-19 — Phase 8 hotfix (`/opt/cloudflare-tunnel`): pre-create Swarm secrets before stack deploy

- **Context:** Після виправлень SOPS decrypt і SSH deploy залишався runtime-fail на кроці `docker stack deploy`: `service tunnel: secret not found: cf_tunnel_token_prod_v1`.
- **Root Cause:** У `scripts/deploy-orchestrator.sh` режим `swarm` рендерив маніфест і одразу виконував `docker stack deploy`, але не виконував `ansible-playbook --tags secrets` перед деплоєм.
- **Change:** У `/opt/cloudflare-tunnel/scripts/deploy-orchestrator.sh` додано функцію `run_ansible_secrets_if_configured`: валідація `INFRA_REPO_PATH`, `ENVIRONMENT_NAME`, наявності `ansible-playbook`, inventory/playbook шляхів (`ansible/inventories/{dev|prod}/hosts.yml`, `ansible/playbooks/swarm.yml`) і запуск `ANSIBLE_CONFIG=<infra>/ansible/ansible.cfg ansible-playbook ... --tags secrets` перед `docker stack deploy`.
- **Verification:** `bash -n /opt/cloudflare-tunnel/scripts/deploy-orchestrator.sh` проходить (`BASH_OK`); diff містить лише очікуване додавання pre-step secrets refresh і виклик цієї функції в `deploy_swarm`.
- **Risks:** Якщо `INFRA_REPO_PATH` не передано або вказано некоректно, скрипт завершується помилкою раніше (fail-fast) і не переходить до `docker stack deploy`.
- **Rollback:** Повернути попередню версію `/opt/cloudflare-tunnel/scripts/deploy-orchestrator.sh` без `run_ansible_secrets_if_configured`.

## 2026-04-19 — Phase 8 hotfix (`/opt/shared-workflows`): forwarding `SOPS_AGE_KEY` to remote orchestrator for `ansible --tags secrets`

- **Context:** Після додавання pre-step `ansible-playbook --tags secrets` у `cloudflare-tunnel` deploy падав на manager з `SOPS payload secrets are configured, but SOPS_AGE_KEY is empty`.
- **Root Cause:** У `shared-ci-cd-swarm.yml` секрет `SOPS_AGE_KEY` використовувався тільки на runner для decrypt `env.*.enc`, але не передавався у remote execution context, де запускається оркестратор і `ansible-playbook`.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` (крок `Deploy on remote host`) додано: 1) `SOPS_AGE_KEY` у `env` кроку; 2) тимчасовий локальний key-файл з `chmod 600`; 3) `scp` key-файлу на remote (`/tmp/sops-age-key.txt`); 4) cleanup key-файлів на runner і remote через `trap`; 5) явну передачу `SOPS_AGE_KEY` в `ORCHESTRATOR_MODE=swarm` процес для коректного Ansible SOPS-flow.
- **Verification:** YAML-парсинг проходить (`YAML_OK`); diff містить лише очікувані зміни у deploy-кроці: env-forwarding ключа, key-file transfer/cleanup, і `SOPS_AGE_KEY=...` при виклику оркестратора.
- **Risks:** `SOPS_AGE_KEY` тимчасово існує у файлі на runner/remote в межах job execution; ризик зменшено `chmod 600`, коротким життєвим циклом і `trap` cleanup.
- **Rollback:** Відкотити зміни у `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` до попередньої ревізії без remote key forwarding.

## 2026-04-19 — Phase 8 step 5 (`/opt/Traefik`): підключено Swarm/SOPS CI path через окремий orchestrator

- **Context:** Після успішної міграції `cloudflare-tunnel` у фазі 8 потрібно було повторити інтеграцію для `/opt/Traefik/` за тією ж моделлю: окремий `deploy-orchestrator-swarm.sh` + dispatcher workflow (`shared-ci-cd.yml`).
- **Change:** У `/opt/Traefik/scripts/deploy-orchestrator-swarm.sh` адаптовано app-специфіку під Traefik: `STACK_NAME` змінено на `traefik`, прибрано cloudflare-специфічну перевірку `CF_TUNNEL_TOKEN_SECRET_NAME`, збережено порядок `ansible-playbook --tags secrets` перед `docker stack deploy`. У `/opt/Traefik/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` встановлено `orchestration_script_path: 'scripts/deploy-orchestrator-swarm.sh'` і `docker_image_name: traefik` (dispatcher `uses: ...shared-ci-cd.yml@main` не змінювався).
- **Verification:** `bash -n /opt/Traefik/scripts/deploy-orchestrator-swarm.sh` проходить без помилок; `python3` (`yaml.safe_load`) успішно парсить `/opt/Traefik/.github/workflows/main.yml` (`YAML_OK`).
- **Risks:** Для коректного remote deploy потрібні налаштовані GitHub Environment Secrets у repo (`SOPS_AGE_KEY`, `SERVER_*`, `DEPLOY_PROJECT_DIR`, за потреби `SERVER_SSH_PORT`); без них job зупиниться на етапі preflight.
- **Rollback:** Повернути у `/opt/Traefik/.github/workflows/main.yml` попередній `orchestration_script_path` та попередню версію `/opt/Traefik/scripts/deploy-orchestrator-swarm.sh`.

## 2026-04-19 — Phase 8 hotfix (`/opt/Traefik`): усунуто падіння `Permission denied` на рендері deploy manifest

- **Context:** На реальному deploy у `scripts/deploy-orchestrator-swarm.sh` падало створення `/tmp/traefik.stack.deploy.yml` з помилкою `Permission denied`.
- **Root Cause:** Скрипт використовував фіксовані шляхи в `/tmp` для `raw/deploy manifest`; при повторних запусках файл міг залишатись із чужим owner/правами, через що shell-redirection не міг перезаписати файл.
- **Change:** У `/opt/Traefik/scripts/deploy-orchestrator-swarm.sh` замінено статичні `/tmp/${STACK_NAME}.stack.*.yml` на унікальні тимчасові файли через `mktemp` у межах `PROJECT_ROOT`; додано `trap 'rm -f ...' RETURN` для гарантованого cleanup при успіху та помилках; явний `rm -f` в кінці функції прибрано як дублюючий.
- **Verification:** `bash -n /opt/Traefik/scripts/deploy-orchestrator-swarm.sh` проходить успішно (`BASH_OK`).
- **Risks:** Якщо файлова система `PROJECT_ROOT` недоступна для запису на manager host, `mktemp` завершиться помилкою раніше (fail-fast) і deploy не стартує.
- **Rollback:** Повернути попередню версію `/opt/Traefik/scripts/deploy-orchestrator-swarm.sh` з фіксованими `/tmp` шляхами (не рекомендовано).

## 2026-04-19 — Phase 8 hotfix (`/opt/Traefik`): приведено `ports[].published` до integer для сумісності з `docker stack deploy`

- **Context:** Після фіксу з тимчасовими файлами deploy падав на валідації Swarm з помилкою `services.traefik.ports.0.published must be a integer`.
- **Root Cause:** `docker compose ... config` рендерив `published` як рядок (`"8080"`), а `docker stack deploy` вимагає integer-тип у deploy-маніфесті.
- **Change:** У `/opt/Traefik/scripts/deploy-orchestrator-swarm.sh` після `awk` додано нормалізацію через `sed -E`, яка перетворює тільки рядки формату `published: "<digits>"` на `published: <digits>` перед `docker stack deploy`.
- **Verification:** `bash -n /opt/Traefik/scripts/deploy-orchestrator-swarm.sh` (`BASH_OK`); тестовий render-пайплайн повертає `published: 8080` без лапок.
- **Risks:** Якщо у майбутньому знадобиться non-numeric вираз у `published`, поточна нормалізація його не змінюватиме (очікувана поведінка).
- **Rollback:** Прибрати доданий `sed -E` етап у `deploy_swarm` і повернути попередній рендер deploy-маніфесту.

## 2026-04-19 — Phase 8 step 6 (`/opt/kdv-integrator/kdv-integrator-event`): виправлено app-ідентифікатори та стабілізовано Swarm orchestrator

- **Context:** Після переходу до наступного стеку знайдено copy-paste залишки від Traefik у KDV repo: некоректні `docker_image_name` та `STACK_NAME`, що не відповідали runbook іменуванню KDV стеку.
- **Change:** У `/opt/kdv-integrator/kdv-integrator-event/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` `docker_image_name` змінено з `traefik` на `kdv-integrator-event`. У `/opt/kdv-integrator/kdv-integrator-event/scripts/deploy-orchestrator-swarm.sh` `STACK_NAME` змінено на `kdv_integrator_event`; також додано патерн стабілізації temp-маніфестів: `mktemp` у `PROJECT_ROOT` + `trap` cleanup.
- **Verification:** `bash -n /opt/kdv-integrator/kdv-integrator-event/scripts/deploy-orchestrator-swarm.sh` (`KDV_BASH_OK`); `yaml.safe_load` для `/opt/kdv-integrator/kdv-integrator-event/.github/workflows/main.yml` (`KDV_YAML_OK`).
- **Risks:** Якщо віддалена директорія repo на manager host недоступна для запису, `mktemp` завершиться помилкою раніше (fail-fast), і deploy буде зупинений до `docker stack deploy`.
- **Rollback:** Повернути попередні значення `docker_image_name`/`STACK_NAME` і прибрати `mktemp+trap` із KDV swarm-скрипта.

## 2026-04-19 — Phase 8 hardening (`/opt/cloudflare-tunnel`): додано `mktemp + trap cleanup` у `deploy-orchestrator-swarm.sh`

- **Context:** Після реального інциденту в Traefik (`Permission denied` на статичному `/tmp/*.stack.deploy.yml`) вирішено уніфікувати захист у вже мігрованому `cloudflare-tunnel`.
- **Change:** У `/opt/cloudflare-tunnel/scripts/deploy-orchestrator-swarm.sh` статичні `/tmp/${STACK_NAME}.stack.raw.yml` і `/tmp/${STACK_NAME}.stack.deploy.yml` замінено на унікальні тимчасові файли через `mktemp` у `PROJECT_ROOT`; додано `trap` для гарантованого cleanup навіть при помилках.
- **Verification:** `bash -n /opt/cloudflare-tunnel/scripts/deploy-orchestrator-swarm.sh` (`CF_BASH_OK`).
- **Risks:** За відсутності прав на запис у `PROJECT_ROOT` на remote host `mktemp` впаде раніше (очікуваний fail-fast).
- **Rollback:** Повернути попередню версію `/opt/cloudflare-tunnel/scripts/deploy-orchestrator-swarm.sh` зі статичними шляхами у `/tmp`.

## 2026-04-20 — Phase 8 step 7 (`/opt/Dspace/DSpace-docker`): вирівняно шаблонні значення після копіювання з KDV

- **Context:** Після успішного deploy `kdv-integrator-event` для наступної ітерації у `DSpace-docker` були перенесені шаблонні файли (`.sops.yaml`, `scripts/deploy-orchestrator-swarm.sh`, `.github/workflows/main.yml`) із KDV repo; частина значень лишилась KDV-специфічною.
- **Change:** У `/opt/Dspace/DSpace-docker/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` `docker_image_name` змінено з `kdv-integrator-event` на `dspace-docker`. У `/opt/Dspace/DSpace-docker/scripts/deploy-orchestrator-swarm.sh` `STACK_NAME` змінено з `kdv_integrator_event` на `dspace` (відповідно до DSpace runbook/stack naming). За уточненням користувача `paths-ignore` не чіпали: актуальна назва файлу — `.env.example`.
- **Verification:** `bash -n /opt/Dspace/DSpace-docker/scripts/deploy-orchestrator-swarm.sh` (`DSPACE_BASH_OK`); `yaml.safe_load` для `/opt/Dspace/DSpace-docker/.github/workflows/main.yml` (`DSPACE_YAML_OK`).
- **Risks:** Якщо у GitHub Environment Secrets для DSpace ще лишилися значення від іншого repo (host/path/secret names), job може впасти на preflight або remote deploy кроках.
- **Rollback:** Повернути попередні значення `docker_image_name` і `STACK_NAME` у файлах DSpace.

## 2026-04-20 — Phase 8 step 8 (`/opt/Koha/koha-deploy`): вирівняно шаблонні значення після копіювання з DSpace

- **Context:** Після переходу до ітерації Koha у repo були перенесені шаблонні файли з DSpace; у результаті в CI залишилися DSpace-специфічні значення для образу та stack name.
- **Change:** У `/opt/Koha/koha-deploy/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` `docker_image_name` змінено з `dspace-docker` на `koha-deploy`. У `/opt/Koha/koha-deploy/scripts/deploy-orchestrator-swarm.sh` `STACK_NAME` змінено з `dspace` на `koha`. Інші вже потрібні для Swarm/SOPS параметри (`use_ansible`, `orchestration_script_path`, `infra_repo_path`, `SOPS_AGE_KEY`, `SERVER_SSH_PORT`) залишено без змін.
- **Verification:** `bash -n /opt/Koha/koha-deploy/scripts/deploy-orchestrator-swarm.sh` (`KOHA_BASH_OK`); `yaml.safe_load` для `/opt/Koha/koha-deploy/.github/workflows/main.yml` (`KOHA_YAML_OK`).
- **Risks:** Якщо GitHub Environment Secrets у Koha environment містять значення від іншого стеку (наприклад, невірний `DEPLOY_PROJECT_DIR` або `SERVER_HOST`), deploy може впасти на remote етапі.
- **Rollback:** Повернути попередні значення `docker_image_name`/`STACK_NAME` у файлах Koha.

## 2026-04-20 — Phase 8 step 9 (`/opt/Matomo-analytics`): вирівняно шаблонні значення після копіювання з Koha

- **Context:** Після переходу до Matomo і переносу шаблонних файлів із Koha у workflow/оркестраторі залишилися Koha-специфічні значення.
- **Change:** У `/opt/Matomo-analytics/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` `docker_image_name` змінено з `koha-deploy` на `matomo-analytics`. У `/opt/Matomo-analytics/scripts/deploy-orchestrator-swarm.sh` `STACK_NAME` змінено з `koha` на `matomo` (відповідно до `docs/deployment.md`).
- **Verification:** `bash -n /opt/Matomo-analytics/scripts/deploy-orchestrator-swarm.sh` (`MATOMO_BASH_OK`); `yaml.safe_load` для `/opt/Matomo-analytics/.github/workflows/main.yml` (`MATOMO_YAML_OK`).
- **Risks:** Якщо Environment Secrets Matomo ще вказують на path/host іншого стеку, deploy зупиниться на preflight/remote етапі.
- **Rollback:** Повернути попередні значення `docker_image_name`/`STACK_NAME` у файлах Matomo.

## 2026-04-20 — Phase 8 step 10 (`/opt/victoriametrics-grafana`): вирівняно шаблонні значення після копіювання з Matomo

- **Context:** Після переходу до `victoriametrics-grafana` та переносу шаблонів із Matomo у workflow і swarm-оркестраторі лишилися Matomo-специфічні значення.
- **Change:** У `/opt/victoriametrics-grafana/.github/workflows/main.yml` для `deploy-dev` і `deploy-prod` `docker_image_name` змінено з `matomo-analytics` на `victoriametrics-grafana`. У `/opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` `STACK_NAME` змінено з `matomo` на `victoriametrics-grafana`.
- **Verification:** `bash -n /opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` (`VMG_BASH_OK`); `yaml.safe_load` для `/opt/victoriametrics-grafana/.github/workflows/main.yml` (`VMG_YAML_OK`).
- **Risks:** За невідповідності GitHub Environment Secrets (наприклад, `DEPLOY_PROJECT_DIR`/`SERVER_HOST`) deploy може впасти на preflight або remote execution кроці.
- **Rollback:** Повернути попередні значення `docker_image_name`/`STACK_NAME` у файлах `victoriametrics-grafana`.

## 2026-04-20 — Phase 8 hotfix (`/opt/victoriametrics-grafana`): idempotent swarm network naming через env (без хардкоду)

- **Context:** Deploy падав на кроці `docker stack deploy` з `failed to create network monitoring_net ... already exists` при вже існуючій мережі `monitoring_net`.
- **Root Cause:** У merged swarm-маніфесті `monitoring_net` успадковувала ім'я з base compose (`MONITORING_NETWORK_NAME=monitoring_net`) і намагалася створювати overlay-мережу з іменем, яке вже зайняте іншою мережею.
- **Change:** У `/opt/victoriametrics-grafana/docker-compose.swarm.yml` для `monitoring_net` додано env-driven `name: ${MONITORING_SWARM_NETWORK_NAME}`. У `/opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` додано обчислення/експорт `MONITORING_SWARM_NETWORK_NAME` (якщо не задано явно): `${STACK_NAME}_${MONITORING_NETWORK_NAME}`. У `/opt/victoriametrics-grafana/.env.example` задокументовано нову змінну `MONITORING_SWARM_NETWORK_NAME`.
- **Verification:** `bash -n /opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` (`VMG_SWARM_SCRIPT_OK`); рендер merge-конфігу показує `monitoring_net -> name: victoriametrics-grafana_monitoring_net` і `driver: overlay`.
- **Risks:** Якщо в Environment Secrets/`env.*.enc` явно задано `MONITORING_SWARM_NETWORK_NAME`, воно має бути унікальним на хості; інакше можливий новий конфлікт імен.
- **Rollback:** Прибрати `MONITORING_SWARM_NETWORK_NAME` з swarm compose/orchestrator і повернути попередню схему іменування мережі.

## 2026-04-20 — Phase 8 hotfix (`/opt/victoriametrics-grafana`): уніфіковано назву swarm-мережі на одну env-змінну `MONITORING_NETWORK_NAME`

- **Context:** За вимогою спрощення конфігурації потрібно прибрати окрему змінну `MONITORING_SWARM_NETWORK_NAME` і керувати назвою мережі тільки через `MONITORING_NETWORK_NAME`.
- **Change:** У `/opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` видалено обчислення/експорт `MONITORING_SWARM_NETWORK_NAME`; додано fail-fast перевірку `MONITORING_NETWORK_NAME` і експорт цієї змінної. У `/opt/victoriametrics-grafana/docker-compose.swarm.yml` `monitoring_net.name` переведено на `${MONITORING_NETWORK_NAME}`. У `/opt/victoriametrics-grafana/.env.example` прибрано дубль і залишено одну канонічну змінну.
- **Verification:** `bash -n /opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` (`VMG_SCRIPT_OK`); merge-render підтверджує `monitoring_net -> name: monitoring_net`, `driver: overlay`.
- **Risks:** Якщо на хості вже існує несумісна мережа з тим самим ім’ям (`monitoring_net`, не overlay), конфлікт при `docker stack deploy` залишиться; у такому разі потрібне унікальне env-значення або cleanup старої мережі.
- **Rollback:** Повернути попередню логіку з окремою змінною `MONITORING_SWARM_NETWORK_NAME`.

## 2026-04-20 — Phase 8 hotfix (`/opt/victoriametrics-grafana`): `MONITORING_NETWORK_NAME` підхоплюється з decrypted env-файлу в CI

- **Context:** Після уніфікації до однієї змінної deploy падав на `ERROR: MONITORING_NETWORK_NAME is not set`, бо у CI значення фактично приходить через `env.dev.enc`/`env.prod.enc` (decrypted у `/tmp/env.decrypted`), а не як shell env змінна процесу.
- **Change:** У `/opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` додано helper `read_env_var_from_file()`, який читає `KEY=VALUE` з `ORCHESTRATOR_ENV_FILE`. Якщо `MONITORING_NETWORK_NAME` не задана у shell env, скрипт тепер підтягує її з `${ENV_FILE}` і тільки потім виконує fail-fast перевірку.
- **Verification:** `bash -n /opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` (`VMG_SCRIPT_OK`).
- **Risks:** Якщо у decrypted env-файлі реально відсутня `MONITORING_NETWORK_NAME`, deploy як і раніше коректно зупиниться на fail-fast перевірці.
- **Rollback:** Видалити helper читання з env-файлу і повернути попередню перевірку тільки shell env.

## 2026-04-20 — Phase 8 hotfix (`/opt/victoriametrics-grafana`): ідемпотентний reuse overlay-мережі `MONITORING_NETWORK_NAME` без падіння на `already exists`

- **Context:** Після релоаду з `MONITORING_NETWORK_NAME=monitoring_net` deploy знову падав на `failed to create network monitoring_net ... already exists`.
- **Root Cause:** `docker stack deploy` намагався створити мережу самостійно; при конфлікті імені це неідімпотентно.
- **Change:** У `/opt/victoriametrics-grafana/docker-compose.swarm.yml` мережу `monitoring_net` переведено в `external: true` з env-іменем `${MONITORING_NETWORK_NAME}`. У `/opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` додано `ensure_swarm_overlay_network()` перед deploy: якщо мережа існує і є `overlay/swarm` — використовуємо її; якщо відсутня — створюємо `docker network create --driver overlay --attachable`; якщо існує, але не `overlay/swarm` — fail-fast із чітким поясненням.
- **Verification:** `bash -n /opt/victoriametrics-grafana/scripts/deploy-orchestrator-swarm.sh` (`VMG_SCRIPT_OK`); merge-render підтверджує `monitoring_net` як `external: true` з `name: monitoring_net`.
- **Risks:** За наявності bridge/non-swarm мережі з тим самим ім'ям deploy коректно зупиниться до ручного вирішення конфлікту (перейменування/видалення старої мережі або зміна env-імені).
- **Rollback:** Повернути попередню схему мережі без `external: true` та прибрати `ensure_swarm_overlay_network()` із оркестратора.

## 2026-04-20 — Phase 8 documentation completion (`/opt/shared-workflows/docs/CI-CD.md`): створено end-to-end CI/CD guide

- **Context:** Після завершення технічної імплементації Phase 8 у всіх 7 stack лишався відкритий блок документації в roadmap (створення `docs/CI-CD.md`, приклади `main.yml`/orchestrator, Environment Secrets setup, troubleshooting).
- **Change:** Створено `/opt/shared-workflows/docs/CI-CD.md` з повною структурою Phase 8: архітектура flow (`dispatcher` -> `compose/swarm`), GitHub Environment Secrets UI setup, приклад per-repo `.github/workflows/main.yml`, template `scripts/deploy-orchestrator-swarm.sh`, manual trigger flow, troubleshooting типових інцидентів (decrypt/recipient/ssh-port/secret-not-found/network-conflict/published-int/permission-denied), security best practices. У `/opt/shared-workflows/docs/ROADMAP.md` відмічено виконаними документаційні пункти розділу `Documentation` та DoD-пункт `docs/CI-CD.md написано`.
- **Verification:** Контент-перевірка нового файлу підтвердила наявність усіх обов'язкових секцій (`GitHub Environment Secrets`, `Per-repo main.yml`, orchestration template, `Troubleshooting`); `CI_CD_LINES=288`.
- **Risks:** При подальших змінах shared workflow (`shared-ci-cd*.yml`) або контрактів orchestration script документ потребуватиме синхронного оновлення, щоб уникнути drift між docs і runtime.
- **Rollback:** Видалити `/opt/shared-workflows/docs/CI-CD.md` і повернути попередні значення чекбоксів у `/opt/shared-workflows/docs/ROADMAP.md`.

## 2026-04-28 — Phase 8 add-on (`/opt/Ansible`): додано delivery-only shared workflow без prod deploy і важких CI checks

- **Context:** Для repo `/opt/Ansible` потрібна не app-деплой логіка, а тільки доставка коду з приватного GitHub repo на remote host; сам repo вже має власний pre-commit lint, тому shared Docker/SOPS/Trivy checks не потрібні.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` додано окремий reusable workflow, який валідовує SSH secrets, опційно підключає Tailscale, виконує remote `git fetch`/`git checkout --detach` у `DEPLOY_PROJECT_DIR` і підтримує `INFRA_REPO_PAT` для приватного GitHub repo без docker build/deploy. У `/opt/Ansible/.github/workflows/main.yml` прибрано copied app-deploy шаблон (`use_ansible`, prod job, SOPS/docker inputs) і залишено один `deliver-dev` job для branch `dev` + manual `workflow_dispatch`.
- **Verification:** `yaml.safe_load` успішно парсить `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` і `/opt/Ansible/.github/workflows/main.yml` (`YAML_OK`).
- **Risks:** На remote host `DEPLOY_PROJECT_DIR` має вже бути git repo з коректним `origin`; для приватного repo потрібен валідний `INFRA_REPO_PAT` або вже налаштований доступ на самому хості.
- **Rollback:** Видалити `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` і повернути попередній `/opt/Ansible/.github/workflows/main.yml`.

## 2026-04-28 — Phase 8 hotfix (`/opt/shared-workflows`): `shared-code-delivery` fetch з приватного GitHub repo через explicit PAT header

- **Context:** Delivery job для `/opt/Ansible` впав на remote `git fetch` з `remote: Write access to repository not granted` / `403`, бо Git міг використовувати credentials із remote `origin`, а не `INFRA_REPO_PAT` через `GIT_ASKPASS`.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` замінено `GIT_ASKPASS git fetch origin` на одноразовий `git fetch` з canonical URL `${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}.git` і `http.extraheader=AUTHORIZATION: basic ...`, сформованим з `INFRA_REPO_PAT`; remote `origin` не змінюється, токен не записується в git config.
- **Verification:** `yaml.safe_load` успішно парсить `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` (`YAML_OK`); `git diff --check` проходить без whitespace-помилок.
- **Risks:** `INFRA_REPO_PAT` має мати доступ на читання до `mzhk-repo/ansible`; для fine-grained PAT потрібні repo access + `Contents: Read`.
- **Rollback:** Повернути попередній блок `GIT_ASKPASS` у `shared-code-delivery.yml`.

## 2026-04-28 — Phase 8 hotfix (`/opt/shared-workflows`): прибрано fallback на `origin` у private repo delivery

- **Context:** Після першого hotfix GitHub Actions продовжив падати з ідентичним `403`, що означало fallback на remote `origin` при порожньому/недоступному `INFRA_REPO_PAT` або неявне використання старих credentials.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` `INFRA_REPO_PAT` зроблено required secret; валідацію secrets доповнено fail-fast перевіркою `INFRA_REPO_PAT`; remote block більше не виконує `git fetch origin`, а завжди fetch-ить приватний repo через explicit PAT header і refspec для heads/tags.
- **Verification:** `yaml.safe_load` успішно парсить `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` (`YAML_OK`); `git diff --check` проходить без whitespace-помилок.
- **Risks:** Якщо `INFRA_REPO_PAT` не заданий або не видимий для environment/job, workflow тепер зупиниться раніше з явною помилкою; якщо PAT заданий, але не має `Contents: Read` до `mzhk-repo/ansible`, GitHub все ще поверне `403`.
- **Rollback:** Повернути `INFRA_REPO_PAT` до optional і відновити fallback `git fetch origin`.

## 2026-04-28 — Phase 8 hotfix (`/opt/shared-workflows`): private repo delivery переведено з remote GitHub fetch на git bundle

- **Context:** Після push змін у `shared-workflows@main` GitHub Actions все ще показував `403` на `https://github.com/mzhk-repo/ansible.git`, тобто будь-який remote fetch із manager host лишався залежним від GitHub credentials/PAT на віддаленому боці.
- **Change:** У `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` delivery-flow переведено на модель `actions/checkout` на runner -> `git bundle create HEAD` -> `scp` bundle на remote host -> `git fetch` з локального bundle-файлу -> `git checkout --detach ${DEPLOY_REF}`. `INFRA_REPO_PAT` знову optional, бо private repo читається runner-ом через стандартний GitHub Actions token; remote host більше не звертається до GitHub.
- **Verification:** `yaml.safe_load` успішно парсить `/opt/shared-workflows/.github/workflows/shared-code-delivery.yml` (`YAML_OK`); `git diff --check` проходить без whitespace-помилок.
- **Risks:** На remote host потрібні `git`, `scp`/SSH доступ і права на запис у `/tmp` для тимчасового bundle; bundle видаляється через `trap` після remote delivery.
- **Rollback:** Повернути попередній remote `git fetch` через GitHub URL і `INFRA_REPO_PAT`.

## 2026-04-29 — Phase 8 hotfix (`/opt/shared-workflows`): додано `safe.directory` guard у Swarm/SOPS remote deploy

- **Context:** GitHub Actions deploy через `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` впав на remote `git fetch` з `fatal: detected dubious ownership in repository`, коли deploy-користувач не був власником директорії repo.
- **Change:** У remote deploy-блоці `shared-ci-cd-swarm.yml` перед `cd "${DEPLOY_PROJECT_DIR}"` і `git fetch --all --prune` додано ідемпотентну перевірку `git config --global --get-all safe.directory`; якщо шлях відсутній, workflow додає `git config --global --add safe.directory "${DEPLOY_PROJECT_DIR}"`.
- **Verification:** `yaml.safe_load` успішно парсить `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` (`YAML_OK`); `git diff --check` проходить без whitespace-помилок.
- **Risks:** Налаштування додається у global git config deploy-користувача на remote host; воно дозволяє Git працювати саме з `DEPLOY_PROJECT_DIR`, але не змінює ownership або filesystem permissions.
- **Rollback:** Прибрати блок `safe.directory` з `shared-ci-cd-swarm.yml`; за потреби вручну видалити запис із global git config deploy-користувача.

## 2026-04-29 — Phase 8 hotfix (`/opt/shared-workflows`): git update у Swarm/SOPS deploy виконується від repo owner при відсутніх write-правах

- **Context:** Після `safe.directory` guard deploy дійшов до реальної файлової помилки: `error: cannot open '.git/FETCH_HEAD': Permission denied`, бо deploy-користувач не мав write-доступу до `.git`.
- **Change:** У remote deploy-блоці `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` додано helper `run_repo_git()`: якщо deploy-користувач має write-права на repo і `.git`, Git виконується напряму; якщо ні, workflow визначає UID власника repo (`stat -c '%u'`) і запускає `git fetch --all --prune` та `git checkout ${DEPLOY_REF}` через `sudo -n -u "#<repo_owner_uid>"`. Якщо passwordless sudo недоступний, job зупиняється з явним поясненням.
- **Verification:** `yaml.safe_load` успішно парсить `/opt/shared-workflows/.github/workflows/shared-ci-cd-swarm.yml` (`YAML_OK`); `git diff --check` проходить без whitespace-помилок.
- **Risks:** Для repo з іншим owner потрібне passwordless sudo для deploy-користувача на виконання git від імені власника repo; workflow не змінює ownership/chmod/ACL автоматично.
- **Rollback:** Прибрати `run_repo_git()` і повернути прямі `git fetch`/`git checkout` від deploy-користувача.
