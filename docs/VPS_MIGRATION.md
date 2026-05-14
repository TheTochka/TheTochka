# Перенос и первичный деплой VPS API

Один и тот же сценарий для **poland** и любого нового сервера. Секреты (SSH, Firebase JSON, Resend, `APP_REQUEST_SECRET`) **не** коммитятся.

## Перед началом

1. DNS: `A` (и при необходимости `AAAA`) для вашего домена (например `poland.vpsperviy.ru`) → IP сервера.
2. JSON сервисного аккаунта Firebase на локальной машине (скачать из Firebase Console).
3. Ключ Resend и случайная строка `APP_REQUEST_SECRET` (та же потом в `--dart-define` Flutter).

---

## Фаза A — снять 3x-ui (если был установлен)

Подключитесь по SSH (ваш порт и пользователь). Дальше — диагностика:

```bash
# Docker
sudo docker ps -a
sudo docker images | grep -iE 'x-ui|3x-ui|xui'

# systemd
systemctl list-units --type=service --all | grep -iE 'x-ui|xui|3x-ui'
ls -la /etc/systemd/system/ | grep -i xui

# Типичные каталоги
ls -la /usr/local/x-ui 2>/dev/null
ls -la /etc/x-ui 2>/dev/null
```

**Если 3x-ui в Docker:**

```bash
sudo docker stop <container_id_or_name>
sudo docker rm <container_id_or_name>
# при необходимости: sudo docker rmi <image_id>
```

**Если сервис systemd (имя может отличаться):**

```bash
sudo systemctl stop x-ui    # или 3x-ui — смотрите list-units
sudo systemctl disable x-ui
sudo rm -f /etc/systemd/system/x-ui.service
sudo systemctl daemon-reload
```

**Скрипт one-line (официальный) часто ставил файлы в `/usr/local/x-ui`:**

```bash
sudo systemctl stop x-ui 2>/dev/null || true
sudo systemctl disable x-ui 2>/dev/null || true
sudo rm -rf /usr/local/x-ui /etc/x-ui
```

Проверьте, что панельные порты свободны: `sudo ss -tlnp`. Закройте лишнее в firewall (оставьте SSH, 80, 443).

---

## Фаза B — Node.js и каталог приложения

```bash
# Debian/Ubuntu: Node 20.x
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

sudo mkdir -p /opt/cross-platform-vps-api
sudo useradd --system --home /opt/cross-platform-vps-api --shell /usr/sbin/nologin vpsapi 2>/dev/null || true
sudo chown vpsapi:vpsapi /opt/cross-platform-vps-api
```

Скопируйте с **локальной** машины (из репозитория) содержимое `backend/vps_api/` **без** `node_modules`:

```bash
# с вашего ПК:
rsync -avz --exclude node_modules \
  ./backend/vps_api/ \
  root@YOUR_SERVER:/opt/cross-platform-vps-api/
```

На сервере:

```bash
cd /opt/cross-platform-vps-api
sudo -u vpsapi npm ci --omit=dev
```

### Автодеплой с Mac / Linux (SSH-ключ)

Пароль в скрипт **не** передаётся: нужен вход по ключу (`ssh-copy-id -p PORT USER@HOST`).

**Новый сервер (например h2.nexus рядом с Польшей):** из корня репозитория можно сгенерировать отдельный ключ и один раз скопировать его на хост.

```bash
# только создать ~/.ssh/id_ed25519_h2-vps{,.pub}
./scripts/bootstrap-ssh-vps.sh --key-only h2-vps

# вариант A — вручную (введёте пароль root один раз)
ssh-copy-id -i ~/.ssh/id_ed25519_h2-vps.pub -p 2222 -o StrictHostKeyChecking=accept-new root@144.31.3.91

# вариант B — macOS + sshpass (пароль только в env, не в истории команд с номером строки — очищайте export после)
brew install hudochenkov/sshpass/sshpass
export SSHPASS='разовый-пароль'
./scripts/bootstrap-ssh-vps.sh --install h2-vps
unset SSHPASS
```

Alias `poland-vps` / `h2-vps` и IP правятся в [`scripts/bootstrap-ssh-vps.sh`](../scripts/bootstrap-ssh-vps.sh) (функция `resolve_host`).

**`~/.ssh/config`** — удобно задать разные ключи для разных VPS:

```
Host poland-vps
  HostName 193.57.137.191
  Port 2222
  User root
  IdentityFile ~/.ssh/id_ed25519_poland-vps
  IdentitiesOnly yes

Host h2-vps
  HostName 144.31.3.91
  Port 2222
  User root
  IdentityFile ~/.ssh/id_ed25519_h2-vps
  IdentitiesOnly yes
```

(Имена ключей должны совпадать с тем, что вы создали: по умолчанию скрипт использует `id_ed25519_<alias>`.)

Из корня репозитория:

```bash
export VPS_HOST=ваш-ip-или-домен
export VPS_PORT=22          # или 2222
export VPS_USER=root
# первый раз: ставит unit и перезапускает сервис
FIRST_RUN=1 ./scripts/deploy-vps-api.sh
# дальше только код (после npm ci по умолчанию: systemctl restart vps-api):
./scripts/deploy-vps-api.sh
# без перезапуска API после выката:
NO_RESTART=1 ./scripts/deploy-vps-api.sh
# timer провижнинга (копирует unit из deploy/, enable + start):
WORKER_TIMER=1 ./scripts/deploy-vps-api.sh
# опционально mock Partner на 127.0.0.1:8790 (smoke/отладка):
MOCK_PARTNER=1 ./scripts/deploy-vps-api.sh
```

Скрипт: [`scripts/deploy-vps-api.sh`](../scripts/deploy-vps-api.sh) (rsync без `.env` и `secrets/` — их создаёте на сервере вручную один раз).

**Аудит после деплоя** (статус unit, `/health`, наличие `.env` и ключа Firebase, Caddy, порты, намёк на 3x-ui):

[`scripts/remote-vps-status.sh`](../scripts/remote-vps-status.sh)

**Деплой кода + аудит одной командой:** [`scripts/vps-deploy-and-status.sh`](../scripts/vps-deploy-and-status.sh)

**Ошибка `ERR: Node.js не найден`** после деплоя — на сервере нет Node. С Mac (после `ssh-copy-id`):

```bash
INSTALL_NODE=1 FIRST_RUN=1 ./scripts/deploy-vps-api.sh
```

(`INSTALL_NODE=1` ставит Node 20 через NodeSource на Debian/Ubuntu.)

---

## Фаза C — секреты и `.env`

```bash
sudo install -o vpsapi -g vpsapi -m 600 /path/from/local/firebase-sa.json \
  /opt/cross-platform-vps-api/secrets/firebase-sa.json
```

Создайте `/opt/cross-platform-vps-api/.env` (права `600`, владелец `vpsapi`). Шаблон: [`backend/vps_api/env.example`](../backend/vps_api/env.example).

Минимум:

```env
PORT=8787
GOOGLE_APPLICATION_CREDENTIALS=/opt/cross-platform-vps-api/secrets/firebase-sa.json
RESEND_API_KEY=re_...
RESEND_MAILBOX=send@yourdomain
APP_REQUEST_SECRET=длинная-случайная-строка
```

```bash
sudo chown vpsapi:vpsapi /opt/cross-platform-vps-api/.env
sudo chmod 600 /opt/cross-platform-vps-api/.env
```

---

## Фаза D — systemd

Скопируйте unit из репозитория:

```bash
sudo cp /opt/cross-platform-vps-api/deploy/vps-api.service /etc/systemd/system/vps-api.service
sudo systemctl daemon-reload
sudo systemctl enable --now vps-api
sudo systemctl status vps-api
```

Проверка локально на сервере: `curl -sS http://127.0.0.1:8787/health` → `ok`.

---

## Фаза E — TLS (Caddy)

Автоматизация (Ubuntu/Debian, под root по SSH): [`scripts/setup-caddy-vps.sh`](../scripts/setup-caddy-vps.sh) — ставит Caddy из официального репозитория и копирует `deploy/Caddyfile.example` в `/etc/caddy/Caddyfile` (заранее поправьте домен в example в репо или на сервере).

Вручную: установите [Caddy](https://caddyserver.com/docs/install), затем:

```bash
sudo cp /opt/cross-platform-vps-api/deploy/Caddyfile.example /etc/caddy/Caddyfile
# отредактируйте домен в Caddyfile при необходимости
sudo systemctl reload caddy
```

Проверка: `curl -sS https://YOUR_DOMAIN/health` → `ok`.

Пример конфига: [`backend/vps_api/deploy/Caddyfile.example`](../backend/vps_api/deploy/Caddyfile.example).

---

## Фаза F — Flutter и Firestore

Сборка/запуск с вашим HTTPS URL:

```bash
flutter run --dart-define=API_BASE_URL=https://YOUR_DOMAIN \
  --dart-define=APP_REQUEST_SECRET=тот-же-что-в-env
```

Убедитесь, что каталог в приложении загружается. **После этого** при необходимости:

```bash
firebase deploy --only firestore:rules
```

(См. [FIREBASE_SERVER_SETUP.md](FIREBASE_SERVER_SETUP.md) — при закрытых rules без работающего API клиент не получит данные.)

---

## Перенос на другой сервер (чеклист)

| Шаг | Действие |
|-----|----------|
| 1 | На старом сервере: `sudo systemctl stop vps-api` (данные Firestore в облаке, локально нужны только `.env` и ключ SA). |
| 2 | Скопировать `/opt/cross-platform-vps-api` **без** `node_modules` **или** заново `rsync` из репо + `npm ci --omit=dev`. |
| 3 | Скопировать `secrets/firebase-sa.json`, `.env`, `/etc/systemd/system/vps-api.service`, фрагмент Caddy. |
| 4 | На новом сервере: DNS A/AAAA → новый IP; повторить фазы B–E. |
| 5 | Обновить `API_BASE_URL` в CI / `--dart-define` релизной сборки. |

**Архив вручную:**

```bash
sudo tar -C /opt -czvf vps-api-backup.tgz \
  --exclude=cross-platform-vps-api/node_modules \
  cross-platform-vps-api
```

Или скрипт из репозитория: [`backend/vps_api/scripts/pack-for-vps.sh`](../backend/vps_api/scripts/pack-for-vps.sh).

---

## Вариант Docker

См. [`backend/vps_api/README.md`](../backend/vps_api/README.md) и `docker-compose.yml` в том же каталоге — перенос сводится к копированию compose, `.env`, volume с ключом и `docker compose up -d` на новом хосте (за reverse-proxy с TLS).
