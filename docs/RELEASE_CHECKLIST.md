# Чеклист релиза (VPS API + клиент)

Перед выкладкой в прод проверьте пункты ниже. Подробнее по деплою сервера: [VPS_MIGRATION.md](VPS_MIGRATION.md).

## Клиент (Flutter)

- **API_BASE_URL** — HTTPS-URL VPS API (например `https://api.example.com`), без завершающего слэша.
- **APP_REQUEST_SECRET** — только если на сервере задан `APP_REQUEST_SECRET`; должен совпадать с заголовком `X-App-Secret` для `POST /api/v1/quote-request`.
- **Не класть секреты в `--dart-define`**: ключи Resend, платёжных провайдеров, сервисного аккаунта Firebase и `PAYMENT_WEBHOOK_SECRET` допустимы **только** на сервере / в секретах CI, не в исходниках и не в артефактах приложения.
- **Чат поддержки:** после обновления **`firestore.rules`** задеплойте их (`firebase deploy --only firestore:rules`), иначе при `USE_SUPPORT_FIRESTORE_LISTENER=true` (по умолчанию) слушатель сообщений получит `permission-denied`. Пока правила не готовы — сборка с `--dart-define=USE_SUPPORT_FIRESTORE_LISTENER=false` (тогда снова опрос VPS раз в `SUPPORT_POLL_INTERVAL_SECONDS`).
- **Опрос VPS (если слушатель чата выключен или для остальных фич):** `CATALOG_POLL_INTERVAL_MINUTES`, `SUPPORT_POLL_INTERVAL_SECONDS`, `REVIEWS_POLL_INTERVAL_MINUTES`, `CLOUD_SYNC_POLL_INTERVAL_MINUTES` — см. `lib/config/app_config.dart`.

## Сервер (VPS API)

- **GOOGLE_APPLICATION_CREDENTIALS** (или `FIREBASE_SERVICE_ACCOUNT_JSON`) — ключ Admin SDK только на машине с API, права файла `600`.
- **RESEND_API_KEY** / **RESEND_MAILBOX** — для запроса КП.
- **APP_REQUEST_SECRET** — опционально, общий секрет с приложением.
- **PAYMENT_WEBHOOK_SECRET** — для `POST /api/v1/webhooks/payment` (заголовок `X-Webhook-Secret`); только на сервере.
- **Health**: `GET /health` или `GET /api/v1/health` → ответ `200` и тело `ok`.

## Firebase

- Задеплоены **firestore.rules** и при необходимости **storage.rules** (`firebase deploy --only firestore:rules,storage`).
- Cloud Functions (агрегаты отзывов): тариф **Blaze**, регион см. [functions/README.md](../functions/README.md) и [FIREBASE_SERVER_SETUP.md](FIREBASE_SERVER_SETUP.md).

## Smoke после деплоя

1. Health снаружи (через тот же домен, что и приложение).
2. Каталог с реального устройства с Bearer-токеном.
3. При включённом секрете — запрос КП с корректным `X-App-Secret`.
4. (Опционально) `POST /api/v1/orders` с Bearer → `GET /api/v1/orders/{id}`; дважды `POST /api/v1/webhooks/payment` с одним `event_id` и `X-Webhook-Secret` — заказ один раз в `paid`.
