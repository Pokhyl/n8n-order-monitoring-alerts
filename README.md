# n8n-order-monitoring-alerts (Portfolio)

Production-grade мониторинг заказов в n8n: SLA-контроль статусов, анти-спам (cooldown), детерминированный лог событий, централизованный Error Handling.

## Что делает
- Принимает вебхук `POST /order-status` с `{ order_id, status, updated_at }`
- Нормализует вход, считает `age_minutes`
- Определяет, нужен ли алерт по SLA:
  - `NEW` слишком долго → `stalled_new` (warn)
  - `PAYMENT_PENDING` слишком долго → `stalled_payment` (critical)
- Формирует `incident_key = order_id + ":" + alert_type`
- Читает состояние инцидента из `incidents` и применяет cooldown (анти-спам)
- Логирует в Google Sheets:
  - `events` — журнал (alert triggered / ignored)
  - `incidents` — текущее состояние по `incident_key` (last_seen/last_alert и т.д.)
  - `error_log` — ошибки через отдельный workflow

## Workflows
- `Order Monitoring (Main)` → `workflows/order-monitoring.sanitized.json`
- `Error Handling (Global)` → `workflows/error-handling.sanitized.json`

## Google Sheets: вкладки и колонки

### events
`event_id, ts, incident_key, order_id, alert_type, severity, status, details_json`

Где `status` (статус алерта): `triggered | ignored`

### incidents
`incident_key, order_id, alert_type, severity, status, first_seen_ts, last_seen_ts, last_alert_ts, last_event_id, details_json`

### error_log
`ts, workflow, execution_id, node, error_message, stack, payload_json`

## Импорт в n8n
1. Import workflow из `workflows/*.sanitized.json`
2. В каждом Google Sheets node:
   - выбрать свой Spreadsheet (Document)
   - выбрать соответствующий Sheet (`events` / `incidents` / `error_log`)
3. В Telegram node (в Error workflow) указать свои credentials и `chatId`
4. В `Order Monitoring (Main)` включить `errorWorkflow` (Settings → Error workflow) и выбрать `Error Handling (Global)`.

## Тестирование
Примеры payload — в папке `samples/`.

Пример curl:
```bash
curl -X POST "https://YOUR_N8N_DOMAIN/webhook-test/order-status" \
  -H "Content-Type: application/json" \
  -d @samples/payload_payment_pending.json
```

## Безопасность
- В репозитории нет секретов: credentials, Telegram chatId, Spreadsheet ID заменены плейсхолдерами.
- Перед деплоем задайте свои credentials в n8n UI.
