# OpenAPI (Swagger) — Partner API

Файл спецификации: [`partner-api.yaml`](partner-api.yaml).

## Как открыть интерактивно

1. **Swagger Editor (онлайн)**  
   Откройте [editor.swagger.io](https://editor.swagger.io), затем **File → Import file** и выберите `partner-api.yaml` из этого каталога (или вставьте содержимое файла).

2. **Локальный предпросмотр**  
   Из корня монорепо, при установленном Node:

   ```bash
   npx --yes @redocly/cli preview-docs docs/openapi/partner-api.yaml
   ```

3. **Swagger UI локально**  
   ```bash
   npx --yes swagger-ui-watcher docs/openapi/partner-api.yaml
   ```

Полное пошаговое руководство для провайдера: [../PARTNER_ONBOARDING.md](../PARTNER_ONBOARDING.md).
