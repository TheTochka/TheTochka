# Partner API — спецификация для провайдеров

**Для владельцев VPN:** пошаговый онбординг — [PARTNER_ONBOARDING.md](PARTNER_ONBOARDING.md). Интерактивная схема (Swagger / OpenAPI) — [openapi/partner-api.yaml](openapi/partner-api.yaml) и [openapi/README.md](openapi/README.md).

Маркетплейс (VPS API) вызывает **ваш** HTTPS-эндпоинт после успешной оплаты заказа. Контракт единый для всех провайдеров; реализация у вас — нативный сервис или [partner-adapter](../partner-adapter/README.md) поверх панели (3XUI, Marzban и т.д.).

Базовый URL и секрет настраиваются в маркетплейсе в Firestore: **`provider_credentials/{providerId}`** (`partnerApiUrl`, `partnerApiToken`) — пишет только Admin SDK.

---

## Аутентификация

Все запросы маркетплейса к Partner API (кроме отдельно оговорённого публичного health для вашего LB):

| Заголовок | Значение |
|-----------|----------|
| `X-Partner-Token` | Секрет провайдера (`sk_live_…`) |
| `Content-Type` | `application/json` (для POST/PATCH) |

Рекомендуемый таймаут на стороне маркетплейса: **10–30 с** (поле `timeoutMs` в `provider_credentials`).

---

## 1. Создание подписки

`POST {baseUrl}/partner/v1/subscription/create`

### Тело запроса

| Поле | Тип | Обязательно | Описание |
|------|-----|---------------|----------|
| `order_id` | string | да | ID заказа в маркетплейсе |
| `tariff_id` | string | да | Согласованный ID тарифа в каталоге |
| `email` | string | да | Email клиента |
| `protocol` | string | нет | Например `vless`, `wireguard` |
| `comment` | string | нет | Метка источника, статистика |

### Успех — HTTP 200, JSON

```json
{
  "status": "success",
  "subscription_id": "sub_xyz789",
  "config": {
    "protocol": "vless",
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "address": "server1.provider.com",
    "port": 443,
    "network": "tcp",
    "security": "reality",
    "sni": "google.com",
    "fingerprint": "chrome",
    "public_key": "xxxx",
    "short_id": "abc",
    "flow": "xtls-rprx-vision"
  },
  "subscription_url": "https://sub.provider.com/...",
  "expires_at": "2025-05-13T00:00:00.000Z"
}
```

Поля `config`, `subscription_url`, `expires_at` желательны: по ним маркетплейс заполняет заказ и письмо клиенту.

**Обратная совместимость:** допускается ответ без обёртки `status`, с полем `id` вместо `subscription_id` (legacy).

### Ошибка — HTTP 200 с `status: "error"` **или** HTTP 4xx/5xx

```json
{
  "status": "error",
  "error_code": "tariff_not_found",
  "message": "Tariff with ID … not found"
}
```

Коды для маппинга: `tariff_not_found`, `capacity_full`, `invalid_request`, `unauthorized`.

---

## 2. Статус подписки

`GET {baseUrl}/partner/v1/subscription/{subscription_id}`

Заголовок `X-Partner-Token`. Ответ — JSON со статусом подписки (формат на усмотрение провайдера, маркетплейс MVP может не вызывать).

---

## 3. Отмена подписки

`DELETE {baseUrl}/partner/v1/subscription/{subscription_id}`

Заголовок `X-Partner-Token`. Успех: HTTP 200, желательно `{ "status": "success" }`.

---

## 4. Health

`GET {baseUrl}/partner/v1/health`

С заголовком `X-Partner-Token` (как в боевых вызовах).

Пример ответа:

```json
{
  "status": "ok",
  "version": "1.0",
  "inbounds_count": 3,
  "active_clients": 150
}
```

---

## 5. Каталог тарифов (синхронизация с маркетплейсом)

`GET {baseUrl}/partner/v1/tariffs`

Заголовок `X-Partner-Token`. Используется **порталом провайдера** на стороне маркетплейса: ответ преобразуется в подколлекцию `providers/{providerId}/tariffs/{id}` в Firestore.

Тело ответа — JSON, массив в поле `tariffs` (или `items`):

```json
{
  "tariffs": [
    {
      "id": "pro_month",
      "name": "Pro 30 дней",
      "days": 30,
      "price_rub": 590,
      "total_gb": 200,
      "limit_ip": 3
    }
  ]
}
```

Поля элементов (гибкая схема, см. маппинг в `backend/vps_api/lib/portal.mjs`):

| Поле | Назначение |
|------|------------|
| `id` / `tariff_id` | ID тарифа в каталоге (совпадает с `tariff_id` в `subscription/create`) |
| `name` | Отображаемое имя |
| `days` | Длительность подписки (для текста в карточке) |
| `price_rub` / `price` | Цена в рублях (строка `price` в каталоге) |
| `total_gb` / `traffic_gb` | Трафик; `0` → «безлимит» в UI по умолчанию |
| `limit_ip` | Число устройств; `0` → «безлимит» |

Опционально: строковые `price`, `traffic`, `devices`, `bandwidth`, `subscription`, `price_period_label`, `included`, `locations` — подставляются в каталог как есть.

Патч для бота: [partner-patches/vpn-bot-stack/marketplace_partner_api.py](../partner-patches/vpn-bot-stack/marketplace_partner_api.py) — сначала вызывается `db_helpers.get_active_tariffs()` (если есть), иначе берётся `TARIFF_MAP` / `TARIFF_MAP_FILE`.

---

## Локальный mock

```bash
cd backend/vps_api
export PARTNER_TOKEN=dev-partner-token
npm run mock-partner
```

По умолчанию `http://127.0.0.1:8790`. Тариф `bad_tariff` / `capacity_full` возвращают `status: error` с соответствующим `error_code`.

---

## Связанные документы

- [DEPLOY_MARKETPLACE_INTEGRATION.md](DEPLOY_MARKETPLACE_INTEGRATION.md) — деплой API, worker, Firestore, приложение.
- [BACKEND_MASTER_PLAN.md](BACKEND_MASTER_PLAN.md) — общая дорожная карта.
- [SECURITY_CHECKLIST.md](SECURITY_CHECKLIST.md) — токены, webhook, прод.
