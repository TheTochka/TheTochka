# Деплой: маркетплейс + провайдеры + приложение

Пошагово: VPS API, Firestore, worker провижнинга, учётные данные Partner, Flutter.

## 1. Firebase

1. Задеплойте правила и индексы (в корне монорепо):

   ```bash
   firebase deploy --only firestore:rules,firestore:indexes
   ```

2. Коллекции создаются при первой записи Admin SDK: `orders`, `webhook_events`, `provider_credentials`, каталог `providers/…`.

## 2. VPS API (systemd)

1. Скопируйте код, `npm install` в `backend/vps_api`.
2. Файл `.env` по [backend/vps_api/env.example](../backend/vps_api/env.example):  
   `GOOGLE_APPLICATION_CREDENTIALS`, `RESEND_API_KEY`, `RESEND_MAILBOX`, при необходимости `APP_REQUEST_SECRET`, `PAYMENT_WEBHOOK_SECRET`, **`ADMIN_API_SECRET`**.
3. Запуск: `node server.mjs` или `npm start`. За reverse-proxy (Caddy) — см. [VPS_MIGRATION.md](VPS_MIGRATION.md).

## 3. Worker провижнинга

Тот же `.env` (или общий файл), **обязательно** `RESEND_API_KEY` / `RESEND_MAILBOX`, если нужны письма после выдачи подписки.

```bash
cd /opt/cross-platform-vps-api
export GOOGLE_APPLICATION_CREDENTIALS=...
node scripts/provision-worker.mjs
```

Рекомендация: **systemd timer** из репозитория (`deploy/provision-worker.timer`) — установка одной командой с машины разработчика: `WORKER_TIMER=1 ./scripts/deploy-vps-api.sh`. Для постоянного цикла вместо timer: `PROVISION_WORKER_LOOP_MS=10000`.

## 4. Учётные данные Partner

Для каждого `providerId` из каталога:

```bash
export GOOGLE_APPLICATION_CREDENTIALS=...
export PROVIDER_ID=direct
export PARTNER_API_URL=https://api.provider.com
export PARTNER_API_TOKEN=sk_live_...
node backend/vps_api/scripts/seed-provider-credentials.mjs
```

Локальная отладка с mock:

```bash
# терминал 1
cd backend/vps_api && PARTNER_TOKEN=secret npm run mock-partner

# терминал 2
export PARTNER_API_URL=http://127.0.0.1:8790
export PARTNER_API_TOKEN=secret
node backend/vps_api/scripts/seed-provider-credentials.mjs
```

На VPS для smoke (mock слушает только localhost): один раз выкатить [`deploy/mock-partner.service`](../backend/vps_api/deploy/mock-partner.service) — `MOCK_PARTNER=1 ./scripts/deploy-vps-api.sh`, затем для каждого `providerId` задать `PARTNER_API_URL=http://127.0.0.1:8790` и токен из `.env` (`PARTNER_TOKEN`, по умолчанию совпадает с `dev-partner-token` из `env.example`). В проде замените URL и токен на реальные Partner API **до** приёма платежей.

## 5. Поток оплаты (сейчас)

1. Приложение: **Оформить заказ** → `POST /api/v1/orders` (Bearer).
2. Оплата: интеграция **platega.io** подключается отдельно; пока тест: `POST /api/v1/webhooks/payment` с `X-Webhook-Secret` и телом:

   ```json
   {
     "event_id": "evt_unique_1",
     "order_id": "<id из шага 1>",
     "status": "paid",
     "payment_id": "pay_xxx",
     "payment_method": "sbp"
   }
   ```

3. Worker забирает `paid` + `provisionQueuedAt` → Partner `POST .../subscription/create` → `active` → письмо клиенту через Resend.

## 6. Flutter

Сборка с базой API:

```bash
flutter run --dart-define=API_BASE_URL=https://ваш-api.example.com
```

Пользователь должен быть **вошедшим в Firebase Auth**. На экране тарифа: **Оформить заказ** → экран статуса с опросом API.

## 7. Проверка

- `GET /health` → `ok`.
- Создать заказ из приложения → webhook (тест) → запустить worker → заказ `active`, письмо на email (если Resend настроен).
