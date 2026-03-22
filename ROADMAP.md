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

/home/pinokew/cloudflare-tunnel

/home/pinokew/Dspace

/home/pinokew/kdv-integrator/kdv-integrator-event

/home/pinokew/Koha/koha-delpoy

/home/pinokew/Matomo-analytics

/home/pinokew/Traefik

/home/pinokew/victoriametrics-grafana

/home/pinokew/Koha/koha-docker-build

[ ] 2.2. Аналіз стеку (для кожної директорії):

Перевірити наявність Dockerfile в корені (це впливатиме на параметр build_and_push_docker).

Перевірити наявність старих конфігурацій у .github/workflows/*.yml.

Визначити ім'я репозиторію на GitHub (з файлу .git/config або командою git remote get-url origin).

Етап 3: Налаштування Environments (GitHub)

Підготовка середовищ на GitHub для кожного цільового репозиторію.

Для КОЖНОГО знайденого репозиторію необхідно виконати :

[ ] 3.1. Створення середовища production (поки пропускаємо):

Створити Environment з назвою production.

Створити секрет DEPLOY_PROJECT_DIR у цьому середовищі. Значення формується на основі назви проєкту (наприклад, /opt/apps/<repo-name>).

[x] 3.2. ✅ Створення середовища development (виконано):

Створити Environment з назвою development.

Створити секрет DEPLOY_PROJECT_DIR для dev-сервера (наприклад, /opt/apps/dev-<repo-name>).


Етап 4: Локальна генерація коду

Фізична зміна файлів на локальному комп'ютері. Коміти та пуші на цьому етапі не виконуються.

Для КОЖНОЇ цільової директорії:

[ ] 4.1. Очищення: Видалити всі існуючі файли конфігурацій CI/CD з папки .github/workflows/ (наприклад, старі ci-cd.yml).

[ ] 4.2. Створення нового пайплайну: Створити новий файл .github/workflows/main.yml.

Шаблон для генерації (Ansible наразі вимкнено):

name: App Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/dev'
    uses: ВАША_ОРГАНІЗАЦІЯ/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: 'development'
      deploy: ${{ github.event_name == 'push' }}
      use_ansible: false # Ansible поки що не використовуємо
      build_and_push_docker: true # Встановити true/false залежно від наявності Dockerfile у проєкті
      docker_image_name: '<назва_репозиторію>'
    secrets: inherit

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    uses: ВАША_ОРГАНІЗАЦІЯ/shared-workflows/.github/workflows/shared-ci-cd.yml@main
    with:
      environment_name: 'production'
      deploy: ${{ github.event_name == 'push' }}
      use_ansible: false # Ansible поки що не використовуємо
      build_and_push_docker: true # Встановити true/false залежно від наявності Dockerfile у проєкті
      docker_image_name: '<назва_репозиторію>'
    secrets: inherit


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