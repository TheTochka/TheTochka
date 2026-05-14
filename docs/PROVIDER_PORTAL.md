# Портал провайдера и автоматизация установки патча

Обновление 2026: портал больше не опирается только на `providerId`-логин. Текущая целевая схема:

- `portal_users/{uid}` — профиль человека в портале.
- `provider_memberships/{providerId__uid}` — роль и доступ человека к provider workspace.
- `portal_staff_memberships/{uid}` — глобальная staff-роль TheTochka + permission set по всей платформе.
- `provider_invites/{inviteId}` — invite-only onboarding.
- `provider_onboarding/{providerId}` — состояние setup wizard.
- `marketplace_finance_settings/default` — глобальная комиссия TheTochka по умолчанию.
- `provider_finance_profiles/{providerId}` — override комиссии, USDT-реквизиты и payout metadata провайдера.
- `provider_payouts/{payoutId}` — ledger выплат провайдерам по периодам.

Идея: **«Портал провайдера»** — веб-вход для мерчантов (логин/пароль), где они задают данные для интеграции с маркетплейсом. Отдельно — **скрипт на их сервере**, который поднимает Partner API (патч) без доступа Польши к root их машины.

---

## 1. Что делать **нельзя** с точки зрения безопасности

**Не хранить на сервере в Польше и не просить в портале:**

- пароль **root** SSH;
- пароли sudo;
- приватные SSH-ключи мерчанта.

Если маркетплейс «сам заходит» на сотни серверов по root-паролю:

- одна утечка БД портала = компрометация **всех** провайдеров;
- высокие юридические и репутационные риски;
- мерчанты с firewalls / только ключи / без пароля root просто не смогут подключиться.

**Вывод:** Польский бэкенд **не** должен массово SSH-иться по паролям на серверы провайдеров. Установка патча остаётся **на стороне мерчанта** (один скрипт от вас), а портал хранит только то, что нужно **Firestore + HTTPS** (публичный URL Partner API, токен, контакты).

---

## 2. Разделение ролей (рекомендуемая модель)

| Где | Что |
|-----|-----|
| **Портал провайдера** (на Польше, позже) | Логин мерчанта → правка **`provider_credentials`**-уровня данных: `partnerApiUrl`, ротация **`partnerApiToken`**, контакты, статусы; опционально просмотр заказов по своему `providerId`. Токен можно **генерировать на сервере** и показывать один раз. |
| **Сервер мерчанта** | Запуск **`bootstrap-partner-api.sh`** (или архива с патчем): копирование файлов в `/root/bot`, `pip`, systemd, напоминание про nginx. **Секреты панели** (Remnawave API token и т.д.) остаются в **его** `.env` бота, в Польшу не отправляются. |
| **Firestore** | Истина для маркетплейса: `provider_credentials/{providerId}`, каталог, заказы. |

---

## 3. Два пути для мерчанта (как сейчас и как улучшить)

### Путь A — только скрипт (уже можно отдавать)

1. Вы выдаёте архив / ссылку на репозиторий с `partner-patches/vpn-bot-stack/`.
2. Мерчант на **своём** сервере: `bash bootstrap-partner-api.sh` (от root, рядом с файлами патча).
3. Скрипт поднимает сервис, печатает **сгенерированный `PARTNER_TOKEN`** и блок для nginx.
4. Мерчант вручную (пока) передаёт вам или в будущем вносит в портал: **URL + токен** → вы пишете в `provider_credentials`.

Это и есть «патч, который они запускают — и всё подключается» **на их машине**; «подключение к маркетплейсу» завершается записью в Firestore (оператор или будущий портал).

### Путь B — Портал провайдера (фазы)

**Фаза 1 (MVP портала):**

- Одна форма после логина: `partnerApiUrl`, `partnerApiToken` (или кнопка «сгенерировать токен»), сохранение в Firestore только для **своего** `providerId` (привязка аккаунта к `providerId` при онбординге оператором).
- Страница «Как установить интеграцию» со ссылкой на `bootstrap-partner-api.sh` и на [MARKETPLACE_USER_FLOW.md](MARKETPLACE_USER_FLOW.md).

**Фаза 2:**

- Список последних заказов по `providerId` (read-only API из VPS).
- Ротация токена, аудит.

**Фаза 3 (опционально, без паролей root):**

- Мерчант добавляет в `authorized_keys` **только публичный ключ** деплоя маркетплейса и явно включает «разрешить деплой»; тогда можно автоматизировать выкладку файлов патча **по ключу**, а не по паролю. Это отдельный жёсткий договор с безопасностью и allowlist IP — не обязательная фича для старта.

---

## 4. Реализация в коде (текущее состояние репозитория)

- Патч Partner API: [partner-patches/vpn-bot-stack/README.md](../partner-patches/vpn-bot-stack/README.md) — в т.ч. `GET /partner/v1/tariffs` для синка тарифов в каталог.
- Автоустановка на сервере мерчанта: **`partner-patches/vpn-bot-stack/bootstrap-partner-api.sh`**.
- **Портал провайдера** встроен в **VPS API** (`backend/vps_api/server.mjs`):
  - Статика: **`GET /portal/`** → `portal_static/index.html`, **`/portal/dashboard.html`**.
  - Новый auth flow: Firebase Auth ID token (`/api/v1/portal/auth/config` + web REST auth) + membership-check на каждом запросе.
  - Firestore: новая identity/access модель —
    - `portal_users/{uid}`
    - `provider_memberships/{providerId__uid}`
  - `portal_staff_memberships/{uid}`
  - `provider_invites/{inviteId}`
  - `provider_onboarding/{providerId}`
  - `marketplace_finance_settings/default`
  - `provider_finance_profiles/{providerId}`
  - `provider_payouts/{payoutId}`
  - `PUT /api/v1/portal/integration` — `provider_credentials` (URL, токен, timeout).
  - `GET /api/v1/portal/onboarding` — wizard state / progress.
  - `GET`/`PUT /api/v1/portal/provider` — поля карточки `providers/{providerId}` (whitelist в `lib/portal.mjs`).
  - `POST /api/v1/portal/sync-tariffs` — `GET {partnerApiUrl}/partner/v1/tariffs`, перезапись подколлекции `tariffs` и обновление `tariffCount` / `priceFrom`.
  - `GET /api/v1/portal/team`, `POST /api/v1/portal/team/invites`, `PUT/DELETE /api/v1/portal/team/members/:uid` — team management.
  - `GET /api/v1/portal/platform/staff`, `POST /api/v1/portal/platform/staff` — глобальные staff-аккаунты TheTochka, роли и permissions.
  - `POST /api/v1/portal/platform/providers` — staff-создание нового провайдера и owner invite без `X-Admin-Secret`.
  - Owner-only backoffice (`platform:manage`):
    - `GET /api/v1/portal/owner/finance/summary` — GMV, комиссия TheTochka, net payout, top providers.
    - `GET /api/v1/portal/owner/finance/sales` — глобальная таблица продаж; query `cursor`, `pageSize` (курсор по `provider_sales.createdAt`).
    - `GET /api/v1/portal/owner/finance/charts` — тайм-серии по выручке, комиссии и payout.
    - `GET`/`PUT /api/v1/portal/owner/finance/settings` — глобальная комиссия по умолчанию.
    - `GET`/`PUT /api/v1/portal/owner/providers/:providerId/finance` — override комиссии + USDT-реквизиты.
    - `GET /api/v1/portal/owner/providers`, `GET /api/v1/portal/owner/providers/:providerId` — глобальный список и карточка провайдера.
    - `GET /api/v1/portal/owner/users`, `GET /api/v1/portal/owner/users/:uid` — глобальный список и карточка пользователя приложения.
    - `GET`/`POST /api/v1/portal/owner/payouts`, `PUT /api/v1/portal/owner/payouts/{payoutId}` — ledger: POST с **детерминированным** id `{providerId}__{periodFrom}__{periodTo}` (нормализованная строка), пересчёт только по продажам с `paymentStatus=paid` и `providerStatus=active`; для статуса `paid` обязателен `reference`.
    - `POST /api/v1/portal/invites/{inviteId}/resend`, `DELETE /api/v1/portal/invites/{inviteId}` — продление invite + письмо, отзыв (`status: revoked`); права: `platform:manage` или admin провайдера для этого `providerId`.
    - Подколлекции **`history`** у `marketplace_finance_settings/default` и `provider_finance_profiles/{providerId}` — append-only снимок при каждом `PUT`.
  - `GET /api/v1/portal/invites/:inviteId`, `POST /api/v1/portal/invites/:inviteId/accept` — invite acceptance flow; при accept роли owner в `provider_onboarding` выставляется `ownerInviteStatus: accepted`.
  - **Письма (Resend):** `lib/resend_portal.mjs` — при создании провайдера, team invite и после upsert staff (без пароля в теле письма). Нужны `RESEND_API_KEY`, `PUBLIC_APP_URL`.
  - `POST /api/v1/admin/providers/onboard` — operator one-shot: create provider + onboarding shell + owner invite.
  - **Нагрузка на Firestore:** в статике портала **нет** фонового опроса (ни `setInterval`, ни авто-рефреша). Запросы к API — только при открытии дашборда (две вкладки данных) и по кнопкам «Сохранить» / «Синхронизировать». Ответы `GET …/integration` и `GET …/provider` **кэшируются в памяти процесса** на время `PORTAL_GET_CACHE_TTL_MS` (по умолчанию 5 минут; `0` — без кэша), сброс кэша после успешного PUT / sync тарифов. Агрегаты owner finance (summary/charts) — отдельный TTL `PORTAL_OWNER_SALES_CACHE_TTL_MS` (по умолчанию 45 с). Таблица продаж в backoffice пагинируется по Firestore (`cursor` + `pageSize` на `GET …/owner/finance/sales`).

### Owner backoffice в UI

Если staff-аккаунт TheTochka имеет permission `platform:manage`, в `portal_static/dashboard.html` доступны два режима сайдбара (**Кабинет провайдера** / **Управление TheTochka**), переключатель хранит выбор в `sessionStorage`.

- **Обзор платформы** — панели «Создать провайдера» и «Команда TheTochka» (раньше были в разделе «Аккаунт»).
- `Финансы` — глобальная сводка, график и таблица продаж (подгрузка страниц по кнопке «Загрузить ещё», если API вернул `nextCursor`).
- `Провайдеры` — KPI по всем провайдерам, детальная карточка, команда, интеграция, USDT-реквизиты.
- `Пользователи` — постраничный список (`cursor`/`pageSize`); на первой странице возможна пометка о частичной статистике по заказам.
- `Выплаты` — ledger + создание snapshot + отметка «Выплачено» через `PUT` с референсом.
- `Финансовые настройки` — глобальная комиссия TheTochka и override на уровне провайдера.

В шапке для операторов показывается предупреждение, если `GET /api/v1/portal/auth/config` вернул `resendConfigured: false` (письма по инвайтам на сервере не настроены).

Даже если у staff-аккаунта **нет** конкретного `provider_membership`, backoffice остаётся доступным: портал не должен показывать пустой экран, а открывает owner-only режим.

### Полезные новые скрипты

- `node scripts/provider-portal-onboard.mjs` — создаёт provider shell, onboarding doc и owner invite вместо ручного набора из нескольких seed-скриптов.
- `node scripts/provider-portal-set-owner.mjs` — жёстко назначает owner-аккаунт провайдера и при желании отзывает остальных owner membership.
- `node scripts/portal-staff-upsert.mjs` — создаёт/обновляет глобальный staff-аккаунт TheTochka в Firebase Auth + Firestore.

### Что проверить после деплоя owner backoffice

1. Войти под `super_owner` / staff-аккаунтом с permission `platform:manage`.
2. Убедиться, что в sidebar видны разделы `Финансы`, `Провайдеры`, `Пользователи`, `Выплаты`, `Финансовые настройки`.
3. Открыть `Финансы` и проверить, что summary / timeline / sales table загружаются без пустого экрана даже при отсутствии выбранного `providerId`.
4. В `Финансовые настройки` сохранить глобальную комиссию и override для одного провайдера, после чего увидеть обновлённый effective percent в карточке провайдера.
5. В `Выплаты` создать тестовый payout и проверить появление записи в `provider_payouts`.

---

## 5. Краткий ответ на вопрос «можем ли мы…»

| Идея | Вердикт |
|------|---------|
| Портал, куда мерчант вводит **root/пароль**, а Польша сама ставит патч | **Не рекомендуем**; не закладываем в архитектуру. |
| Портал для **URL + Partner token + инструкций** | **Да**, это правильное направление. |
| **Один скрипт** у мерчанта, который всё поднимает на **его** сервере | **Да**, см. `bootstrap-partner-api.sh`. |
| Авто-деплой с Польши по **SSH-ключу** и явному согласию | Возможно позже как фаза 3, не как хранение root-пароля. |
