# Настройка Firebase + VPS API (полный прокси)

Полный чеклист установки на VPS, снятие 3x-ui и перенос на другой сервер: **[VPS_MIGRATION.md](VPS_MIGRATION.md)**.

## 1. Сервисный аккаунт

1. Откройте [Firebase Console](https://console.firebase.google.com) → ваш проект → **Project settings** → **Service accounts**.
2. Нажмите **Generate new private key** (или используйте существующий ключ Admin SDK).
3. Сохраните JSON **только на VPS** (не в репозиторий). Права файла: `chmod 600`.
4. На сервере задайте:
   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS=/var/secrets/firebase-sa.json
   ```
   Альтернатива: переменная `FIREBASE_SERVICE_ACCOUNT_JSON` с **сырым JSON** (удобно в PaaS; храните в секретах платформы).

**Роли в GCP:** для стандартного ключа из вкладки Firebase отдельная настройка IAM обычно не нужна. Если создаёте свой SA вручную, выдайте доступ к Firestore согласно документации Google Cloud (Datastore/Firestore User или роли уровня проекта по политике безопасности).

## 2. Закрытие Firestore для клиента

В репозитории файл `firestore.rules` запрещает любые `read/write` с мобильного SDK. Данные пишет только:

- **VPS** (Admin SDK),
- **Cloud Functions** (например `syncReviewAggregates`).

Деплой правил:

```bash
firebase deploy --only firestore:rules
```

**Важно:** сначала поднимите VPS API и проверьте каталог с приложения, затем деплойте правила. Иначе клиент потеряет доступ к данным.

**Квота Firestore (VPS):** сервер кэширует в памяти процесса:
- **`GET /api/v1/catalog`** — `CATALOG_CACHE_TTL_MS` (по умолчанию **10 мин**); при обновлении каталога из портала кэш сбрасывается.
- **`GET /api/v1/me/sync`** — `CLOUD_SYNC_CACHE_TTL_MS` (по умолчанию **4 мин**); сброс после `PUT …/sync/settings` или `…/favorites`.
- **`provider_credentials`** в **provision-worker** (отдельный процесс) — `PROVIDER_CREDENTIALS_CACHE_TTL_MS` (по умолчанию **5 мин**): меньше `get` на каждый заказ того же провайдера. После смены URL/токена в портале worker подхватит их не позже этого TTL (или `systemctl restart provision-worker`).
- Админ-метрики — `ADMIN_METRICS_CACHE_TTL_MS` (по умолчанию 2 мин).

Каталог **не** подгружает подколлекцию `reviews` в этот ответ — в карточке используйте **`rating`** и **`reviewCount`** на документе `providers/{id}`. Отзывы списком — `GET /api/v1/providers/{id}/reviews`.

**Портал провайдера и `RESOURCE_EXHAUSTED`:** новый provider portal больше не читает `provider_portal_users` при логине, потому что вход идёт через Firebase Auth ID tokens. Но квота Firestore всё ещё важна для membership/onboarding/integration запросов. Если в Firebase Console → **Firestore → Usage** видно превышение бесплатной квоты (или проект на Spark и дневной лимит исчерпан), portal API начнёт падать уже на чтениях `portal_users`, `provider_memberships`, `portal_staff_memberships`, `provider_onboarding`, `providers` и `provider_credentials`. Решение: держать проект на **Blaze**, использовать кэш и не плодить лишние auto-refresh сценарии.

**Чат поддержки:** при `USE_SUPPORT_FIRESTORE_LISTENER=true` (по умолчанию в `AppConfig`) список сообщений идёт через **Firestore `snapshots()`** на `support_chats/{uid}/messages` — обновления при новых документах, **без** периодического GET к VPS. Отправка текста/фото по-прежнему **`POST /api/v1/me/support/messages`** на VPS + Storage. Нужны правила из репозитория (`allow read` только если `request.auth.uid == uid` на эту подколлекцию). Отключить слушатель и вернуться к опросу VPS: `--dart-define=USE_SUPPORT_FIRESTORE_LISTENER=false`.

Отзывы: приложение шлёт **`?after=…`** — VPS читает инкрементально. При запросе Firebase может попросить **составной индекс** по `createdAt` — ссылка в логах консоли.

## 3. Storage

Правила `storage.rules` без изменений: пользователь с `request.auth.uid` может грузить файлы в `support_chats/{uid}/…`. Метаданные сообщений пишет VPS в Firestore.

## 4. Опциональный документ секретов в Firestore

Создайте коллекцию `_config`, документ `runtime_secrets` (путь как в `FIRESTORE_SECRETS_PATH`). Поля (пример):

```json
{
  "resendApiKey": "re_...",
  "resendMailbox": "send@example.com",
  "appRequestSecret": "длинная-случайная-строка",
  "paymentWebhookSecret": "секрет-для-x-webhook-secret-mvp"
}
```

Создавайте документ **в консоли** или скриптом с Admin SDK. В правилах Firestore клиент к нему не имеет доступа.

### 4.1 Учётные данные Partner (`provider_credentials`)

Документы **`provider_credentials/{providerId}`** (id как у провайдера в каталоге): поля `partnerApiUrl`, `partnerApiToken`, опционально `integrationStatus`, `timeoutMs`. Пишет только Admin SDK; см. [ADR 0002](adr/0002-provider-credentials-firestore.md), скрипт `backend/vps_api/scripts/seed-provider-credentials.mjs`.

## 5. Cloud Functions: регион и Blaze

Функция **syncReviewAggregates** в коде задана в регионе **`europe-west1`** (см. [`functions/index.js`](../functions/index.js)). Для `firebase deploy --only functions` в проекте должен быть подключён платёжный аккаунт и тариф **Blaze**. Кратко: [functions/README.md](../functions/README.md).

## 6. Запуск API на VPS

См. `backend/vps_api/README.md`: `npm install`, env, `node server.mjs` или `npm start`.

Рекомендуется процесс под systemd с `Restart=always` и reverse-proxy (HTTPS) на порт приложения.

## 7. Проверка

1. `curl https://api.ваш-домен/health` → `ok`.
2. Получите ID token (из тестового приложения или Firebase Auth REST) и выполните:
   ```bash
   curl -H "Authorization: Bearer <token>" https://api.ваш-домен/api/v1/catalog
   ```
3. В приложении: `--dart-define=API_BASE_URL=https://api.ваш-домен`.

## 8. Минимальные права (на будущее)

Для жёсткой модели zero-trust можно сузить SA до операций только Firestore/Auth; это отдельная задача в IAM. На старте достаточно ключа из Firebase Console при условии, что ключ не утекает и ротируется при компрометации.
