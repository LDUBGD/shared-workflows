# SSH deploy user runbook for GitHub Actions

Цей ранбук описує, як підготувати нового користувача на сервері для deploy через SSH з reusable GitHub Actions workflow `shared-ci-cd.yml`, `shared-ci-cd-compose.yml` або `shared-ci-cd-swarm.yml`.

## 1) Що саме налаштовуємо

GitHub Actions підключається до remote host напряму:

```text
GitHub Actions runner
  -> ssh -p <SERVER_SSH_PORT> <SERVER_USER>@<SERVER_HOST>
  -> scp/ssh deploy steps
```

У цьому flow є два різні типи SSH-ключів:

| Дані | Що це | Де зберігається |
|---|---|---|
| `SERVER_SSH_KEY` | Private key deploy-користувача, яким GitHub Actions логіниться на сервер | GitHub Environment Secret |
| `authorized_keys` | Public key від `SERVER_SSH_KEY` | На сервері: `/home/<SERVER_USER>/.ssh/authorized_keys` |
| `SSH_HOST_KEY_PUB` | Public host key самого сервера | GitHub Environment Secret |
| `/etc/ssh/ssh_host_ed25519_key.pub` | Public host key самого сервера | На сервері |

Важливо: `SSH_HOST_KEY_PUB` не має збігатися з ключем користувача. Це ключ сервера, а не deploy-користувача.

## 2) Підготувати deploy-користувача на сервері

На prod/dev сервері створити або вибрати користувача, під яким workflow має виконувати deploy. Приклад нижче використовує `deploy`.

```bash
sudo adduser deploy
```

Якщо користувач уже існує, цей крок пропустити.

Створити SSH-директорію:

```bash
sudo mkdir -p /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chown deploy:deploy /home/deploy/.ssh
```

Переконатися, що користувач має потрібні права для deploy:

```bash
sudo usermod -aG docker deploy
```

Якщо orchestration script виконує `sudo`, додати окреме правило sudoers тільки для потрібних команд. Не давати ширші права без потреби.

## 3) Створити окремий CI deploy key без passphrase

Для GitHub Actions краще створювати окремий ключ, не використовувати особистий SSH key розробника.

На локальній машині або на адміністративному хості:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_actions_prod_deploy -C "github-actions-prod-deploy"
```

Коли `ssh-keygen` попросить passphrase, натиснути Enter двічі, щоб залишити passphrase порожнім.

Причина: у workflow SSH запускається з `BatchMode=yes`, тому GitHub Actions не може інтерактивно ввести passphrase. Якщо key має passphrase, deploy впаде з:

```text
Permission denied (publickey).
scp: Connection closed
```

## 4) Додати public key користувача на сервер

Вивести public key:

```bash
cat ~/.ssh/github_actions_prod_deploy.pub
```

Додати цей рядок на сервері у `authorized_keys` deploy-користувача:

```bash
sudo nano /home/deploy/.ssh/authorized_keys
```

Після редагування виставити права:

```bash
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
```

Перевірити локально:

```bash
ssh -i ~/.ssh/github_actions_prod_deploy -p <SERVER_SSH_PORT> deploy@<SERVER_HOST>
```

Очікувано: підключення проходить без `Enter passphrase for key` і без `Permission denied`.

## 5) Отримати host key сервера

`SSH_HOST_KEY_PUB` має бути public host key сервера для конкретного host і port, до яких підключається GitHub Actions.

З довіреної машини:

```bash
ssh-keyscan -p <SERVER_SSH_PORT> -t ed25519 <SERVER_HOST>
```

Приклад виводу:

```text
[100.93.26.123]:22222 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK91sLFuHgxaei068t59VQvKqwwrvjtDfNvLJ1IHMSsQ
```

У GitHub secret `SSH_HOST_KEY_PUB` вставляти тільки тип і сам ключ:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK91sLFuHgxaei068t59VQvKqwwrvjtDfNvLJ1IHMSsQ
```

На сервері той самий key зазвичай можна побачити так:

```bash
sudo cat /etc/ssh/ssh_host_ed25519_key.pub
```

Якщо `ssh-keyscan` і `/etc/ssh/ssh_host_ed25519_key.pub` не збігаються, перевірити, який SSH daemon слухає потрібний порт:

```bash
sudo sshd -T -p <SERVER_SSH_PORT> | grep hostkey
sudo grep -R "^HostKey" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*
```

## 6) Перевірити fingerprint host key

Перед тим як зберігати `SSH_HOST_KEY_PUB`, перевірити fingerprint:

```bash
ssh-keyscan -p <SERVER_SSH_PORT> -t ed25519 <SERVER_HOST> | ssh-keygen -lf -
```

Приклад:

```text
256 SHA256:jgeqeHFseoVVMt7gO4qm9enJZoZrDjsiNLHSHHD7k1Y [100.93.26.123]:22222 (ED25519)
```

Цей fingerprint має збігатися з тим, який показує SSH/GitHub Actions для цього host.

## 7) Додати GitHub Environment Secrets

У потрібному GitHub Environment (`development`, `production`) додати:

| Secret | Значення |
|---|---|
| `SERVER_HOST` | IP/DNS/Tailscale hostname сервера, наприклад `100.93.26.123` |
| `SERVER_SSH_PORT` | SSH port, наприклад `22222`; якщо port `22`, можна не задавати |
| `SERVER_USER` | Deploy-користувач на сервері, наприклад `deploy` |
| `SERVER_SSH_KEY` | Повний private key з `~/.ssh/github_actions_prod_deploy` |
| `SSH_HOST_KEY_PUB` | Public host key сервера у форматі `ssh-ed25519 AAAA...` |
| `DEPLOY_PROJECT_DIR` | Шлях до repo на сервері, наприклад `/opt/my-app` |

`SERVER_SSH_KEY` має містити весь private key:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

## 8) Як workflow використовує ці secrets

Reusable workflow створює `~/.ssh` на GitHub runner, а не на сервері:

```bash
install -m 700 -d ~/.ssh
printf '%s\n' "$SERVER_SSH_KEY" > ~/.ssh/id_ed25519
printf '[%s]:%s %s\n' "$SERVER_HOST" "$SERVER_SSH_PORT" "$SSH_HOST_KEY_PUB" > ~/.ssh/known_hosts
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/known_hosts
```

Після цього deploy step виконує:

```bash
scp -P "${SERVER_SSH_PORT}" -o BatchMode=yes -o StrictHostKeyChecking=yes ...
ssh -p "${SERVER_SSH_PORT}" -o BatchMode=yes -o StrictHostKeyChecking=yes ...
```

Тому:

- `known_hosts` на сервері не потрібен для входу GitHub Actions на сервер;
- `authorized_keys` на сервері потрібен;
- private key з passphrase не підходить для цього workflow;
- `SSH_HOST_KEY_PUB` має відповідати саме `SERVER_HOST` + `SERVER_SSH_PORT`.

## 9) Troubleshooting

### `REMOTE HOST IDENTIFICATION HAS CHANGED`

Причина: `SSH_HOST_KEY_PUB` у GitHub secrets не відповідає host key сервера для поточного `SERVER_HOST` і `SERVER_SSH_PORT`.

Що зробити:

```bash
ssh-keyscan -p <SERVER_SSH_PORT> -t ed25519 <SERVER_HOST>
ssh-keyscan -p <SERVER_SSH_PORT> -t ed25519 <SERVER_HOST> | ssh-keygen -lf -
```

Оновити `SSH_HOST_KEY_PUB` актуальним `ssh-ed25519 ...`.

### `Permission denied (publickey)` локально

Причина: public key від private key не доданий у `authorized_keys` потрібного користувача або доданий не тому користувачу.

Що зробити:

```bash
ssh-keygen -y -f ~/.ssh/github_actions_prod_deploy
sudo nano /home/<SERVER_USER>/.ssh/authorized_keys
sudo chmod 700 /home/<SERVER_USER>/.ssh
sudo chmod 600 /home/<SERVER_USER>/.ssh/authorized_keys
sudo chown -R <SERVER_USER>:<SERVER_USER> /home/<SERVER_USER>/.ssh
```

### Локально працює, але просить `Enter passphrase for key`

Причина: private key зашифрований passphrase. GitHub Actions не зможе його використати у цьому workflow, бо SSH запускається з `BatchMode=yes`.

Що зробити: створити окремий CI deploy key без passphrase і додати його public key у `authorized_keys`.

### Локально працює, а GitHub Actions все одно показує `Permission denied (publickey)`

Перевірити:

- `SERVER_USER` збігається з користувачем, у якого оновлено `authorized_keys`;
- `SERVER_SSH_KEY` містить private key, а не `.pub`;
- private key вставлений повністю, включно з `BEGIN`/`END`;
- public key від `SERVER_SSH_KEY` є в `/home/<SERVER_USER>/.ssh/authorized_keys`;
- на сервері правильні permissions для `.ssh` і `authorized_keys`;
- `SERVER_HOST` і `SERVER_SSH_PORT` ведуть саме на той сервер, де оновлено `authorized_keys`.

## 10) Швидкий checklist

- [ ] На сервері є deploy-користувач: `<SERVER_USER>`.
- [ ] У `/home/<SERVER_USER>/.ssh/authorized_keys` додано public key від CI deploy key.
- [ ] Private key CI deploy key не має passphrase.
- [ ] Локальна перевірка `ssh -i <key> -p <port> <user>@<host>` проходить без prompt.
- [ ] `SSH_HOST_KEY_PUB` взято через `ssh-keyscan -p <port> -t ed25519 <host>`.
- [ ] Fingerprint host key перевірено через `ssh-keygen -lf -`.
- [ ] У GitHub Environment Secrets задано `SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`, `SSH_HOST_KEY_PUB`, `DEPLOY_PROJECT_DIR`.
- [ ] Якщо SSH port не `22`, задано `SERVER_SSH_PORT`.
