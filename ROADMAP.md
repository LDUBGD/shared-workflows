🗺️ Дорожня карта: Міграція на Shared CI/CD

Мета: Проаналізувати вказані локальні директорії репозиторіїв, підготувати для них GitHub Environments та локально згенерувати/оновити файли CI/CD для подальшого ручного коміту та пушу.

Етап 1: Валідація глобального стану (Організація та Центральний репо)

Перевірка базової інфраструктури перед початком масових змін.

[x] 1.1. Перевірка центрального репо: Переконатися, що в репозиторії з шаблонами наявні лише shared-ci-cd.yml та ROADMAP.md.

[x] 1.2. Аудит Organization Secrets (Спільний сервер): Оскільки зараз усі проєкти хостяться на одному сервері, перевірити наявність таких секретів на рівні Організації (в налаштуваннях GitHub):

TS_OAUTH_CLIENT_ID

TS_OAUTH_SECRET

SERVER_HOST (Спільний IP)

SERVER_USER (Спільний юзер)

SERVER_SSH_KEY (Спільний ключ)

SSH_HOST_KEY_PUB

💡 Архітектурна примітка (Future-proof): Коли в майбутньому проєкти переїдуть на власні сервери, достатньо буде створити секрети з такими ж іменами (SERVER_HOST, SERVER_SSH_KEY тощо) у розділі Environments конкретного репозиторію. GitHub Actions автоматично перевизначить (override) організаційний секрет на секрет середовища. Код пайплайну при цьому змінювати не потрібно.

Етап 2: Фаза Discovery (Локальний аудит)

Збір контексту з локальних директорій проєктів.

[ ] 2.1. Цільові директорії: Опрацювати наступний список локальних шляхів по черзі, не всі одразу:

✅ /home/pinokew/cloudflare-tunnel

✅ /home/pinokew/Dspace/DSpace-docker

✅ /home/pinokew/Koha/koha-deploy

✅ /home/pinokew/Matomo-analytics

✅ /home/pinokew/Traefik

✅ /home/pinokew/victoriametrics-grafana

✅ /home/pinokew/kdv-integrator/kdv-integrator-event

/home/pinokew/Koha/koha-docker-build

[ ] 2.2. Аналіз стеку (для кожної директорії):

Перевірити наявність Dockerfile в корені (це впливатиме на параметр build_and_push_docker).

Перевірити наявність старих конфігурацій у .github/workflows/*.yml.

Перед видаленням старих workflow обов'язково проаналізувати `.github/workflows/` на наявність існуючих патернів, кастомних умов та скриптів перевірки/деплою, які мають бути перенесені в локальний `main.yml`.

Окремо проаналізувати наявність pre-deploy/post-deploy скриптів і сформувати/оновити єдиний repo-specific скрипт оркестрації (рекомендований шлях: `scripts/deploy-orchestrator.sh`).

Визначити ім'я репозиторію на GitHub (з файлу .git/config або командою git remote get-url origin).

Етап 3: Налаштування Environments (GitHub)

Підготовка середовищ на GitHub для кожного цільового репозиторію.

Для КОЖНОГО знайденого репозиторію необхідно виконати :

[х] 3.1. ✅ Створення середовища production (виконано):

Створити Environment з назвою production.

Створити секрет DEPLOY_PROJECT_DIR у цьому середовищі. Значення формується на основі назви проєкту (наприклад, /opt/apps/<repo-name>).

[x] 3.2. ✅ Створення середовища development (виконано):

Створити Environment з назвою development.

Створити секрет DEPLOY_PROJECT_DIR для dev-сервера (наприклад, /opt/apps/dev-<repo-name>).


Етап 4: Локальна генерація коду

Фізична зміна файлів на локальному комп'ютері. Коміти та пуші на цьому етапі не виконуються.

Для КОЖНОЇ цільової директорії:

[ ] 4.1. Аналіз перед очищенням: Перед видаленням старих workflow зафіксувати потрібні патерни/кастомні кроки/опціональні скрипти з `.github/workflows/` конкретного репо для перенесення в новий локальний `main.yml`.

[ ] 4.2. Очищення: Після аналізу видалити застарілі файли CI/CD з `.github/workflows/` (наприклад, старі `ci-cd.yml`).

[ ] 4.3. Створення нового пайплайну: Створити новий `.github/workflows/main.yml` на базі шаблону `/home/pinokew/shared-workflows/main.example.yml` з урахуванням виявлених локальних патернів і опціональних скриптів.

Логіка запуску середовищ (оновлено):

- development:
  - PR у гілку dev: запускаються тільки CI-перевірки.
  - Push (merge) у гілку dev: запускаються CI-перевірки + деплой у development.

- production:
  - PR у гілку main (включно з PR dev -> main): запускаються тільки CI-перевірки.
  - Push (merge) у гілку main: запускаються CI-перевірки без деплою.
  - CD у production виконується тільки при створенні релізу з тегом формату vx.x.x (наприклад, v1.2.3).

Шаблон для генерації (Ansible наразі вимкнено):

name: App Pipeline

on:
  push:
    branches: [ dev, main ]
  pull_request:
    branches: [ dev, main ]
  release:
    types: [ published ]

jobs:
  deploy-dev:
    if: github.event_name == 'pull_request' && github.base_ref == 'dev' || github.event_name == 'push' && github.ref == 'refs/heads/dev'
    uses: ВАША_ОРГАНІЗАЦІЯ/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: 'development'
      deploy: ${{ github.event_name == 'push' && github.ref == 'refs/heads/dev' }}
      use_ansible: false # Ansible поки що не використовуємо
      build_and_push_docker: true # Встановити true/false залежно від наявності Dockerfile у проєкті
      docker_image_name: '<назва_репозиторію>'
    secrets: inherit

  deploy-prod:
    if: github.event_name == 'pull_request' && github.base_ref == 'main' || github.event_name == 'push' && github.ref == 'refs/heads/main' || github.event_name == 'release' && startsWith(github.event.release.tag_name, 'v')
    uses: ВАША_ОРГАНІЗАЦІЯ/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: 'production'
      deploy: ${{ github.event_name == 'release' && startsWith(github.event.release.tag_name, 'v') }}
      use_ansible: false # Ansible поки що не використовуємо
      build_and_push_docker: true # Встановити true/false залежно від наявності Dockerfile у проєкті
      docker_image_name: '<назва_репозиторію>'
    secrets: inherit

Примітка: для суворої перевірки саме формату vx.x.x варто додати regex-перевірку тега (наприклад, ^v[0-9]+\.[0-9]+\.[0-9]+$) у кроці валідації всередині пайплайну.

Важливо: repo-specific кроки (`verify-env.sh`, `patch-local.cfg.sh`, `init-volumes.sh` та інші) потрібно консолідувати в єдиний скрипт оркестрації (рекомендовано `scripts/deploy-orchestrator.sh`), який викликається через `shared-ci-cd.yml` опційно.

Оновлена модель опціональних скриптів: локальний `main.yml` не запускає repo-specific скрипти окремими job (щоб не хардкодити тулзи типу `actions/checkout`), а лише передає `orchestration_script_path` у reusable workflow. У `shared-ci-cd.yml` цей скрипт викликається опційно (тільки якщо файл існує), інакше крок пропускається.

Правило централізації тулзів: усі версії GitHub Actions/CI тулзів (`actions/checkout`, `docker/*`, `tailscale/*`, `trivy`, `gitleaks` тощо) мають керуватися в `shared-workflows`; у локальних репозиторіях не хардкодимо ці тулзи, локальний `main.yml` лише оркеструє виклики reusable workflow з `shared-workflows`.

Правило синхронізації шаблону: якщо змінюємо будь-яку логіку в локальному `.github/workflows/main.yml`, таку саму зміну потрібно внести і в `/home/pinokew/shared-workflows/main.example.yml`.


Етап 5: Ручні дії та фіналізація

Завершальний етап міграції.

[ ] 5.1. Фінальна перевірка: Переконатися, що у відповідних репозиторіях на GitHub створено середовища та секрети DEPLOY_PROJECT_DIR, а локально згенеровано нові файли main.yml.

[ ] 5.2. Коміт та пуш: Відкрити термінал, зайти в кожну опрацьовану директорію зі списку та виконати наступні команди:

git add .github/workflows/
git commit -m "chore: migrate to shared CI/CD"
git push


🛠️ Технічні примітки для автоматизації:

Ізоляція змін: Будь-які скрипти автоматизації мають лише модифікувати файли. Виконання git commit або git push повинно залишатися суворо ручним кроком.

Скрипт перевірки типу: ./scripts/verify-env.sh і т.д. для кожно репу повинні бути опційними, повинна бути можливість добавляти унікальні скрипти для кожного репо