# Безопасность бэкенда (чеклист по §13 BACKEND_MASTER_PLAN)

Отметьте пункты при аудите. Детали деплоя: [RELEASE_CHECKLIST.md](RELEASE_CHECKLIST.md), [FIREBASE_SERVER_SETUP.md](FIREBASE_SERVER_SETUP.md).

- [ ] **Секреты** не в репозитории и не в клиентском бинарнике; ротация ключей описана и выполняется (Resend, webhook, admin, Partner token).
- [ ] **Webhook оплаты:** в проде — проверка подписи платёжной системы и сырой body для HMAC; MVP `X-Webhook-Secret` только для staging/внутренних тестов.
- [ ] **Rate limit** на публичные эндпоинты (КП, создание заказа) по IP и/или по пользователю — на стороне reverse-proxy (Caddy/nginx) или отдельным слоём (не всё может быть в Node).
- [ ] **Partner token** в проде хранить шифрованием (KMS / Secret Manager); в Firestore `provider_credentials` только через Admin SDK, клиентский SDK к коллекции не ходит ([firestore.rules](../firestore.rules)).
- [ ] **Выдача VPN-конфига** пользователю — по HTTPS; ссылки в письмах — осознанный TTL и политика.
- [ ] **Сервис-аккаунт** Firebase на VPS с минимальными IAM-правами (см. [FIREBASE_SERVER_SETUP.md](FIREBASE_SERVER_SETUP.md) §8).
- [ ] **Admin API** (`X-Admin-Secret`) не светить в логах клиентов; длинный случайный секрет, только на сервере.
- [ ] **Алерты** (фаза 6): рост `failed` заказов, недоступность Partner health — настроить в мониторинге инфраструктуры по JSON-логам или метрикам.
