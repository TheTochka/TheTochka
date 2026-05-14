# ADR 0002: учётные данные Partner в Firestore

## Статус

Принято (MVP).

## Контекст

VPS API должен вызывать Partner API провайдера (`partnerApiUrl`, секрет). Секреты не должны попадать в клиент и не должны лежать в репозитории.

## Решение

- Коллекция **`provider_credentials`**, документ **`{providerId}`** (совпадает с id провайдера в каталоге).
- Поля: `partnerApiUrl`, `partnerApiToken`, опционально `integrationStatus` (`active` | `testing` | `disabled`), `timeoutMs`.
- Доступ только **Admin SDK**; в [firestore.rules](../firestore.rules) — запрет для клиента.

## Последствия

- В MVP токен хранится **в открытом виде** в Firestore (как и прочие секреты в `_config/runtime_secrets` на старте). Для продакшена планируется шифрование или вынесение в Secret Manager (см. [SECURITY_CHECKLIST.md](../SECURITY_CHECKLIST.md)).
