# n8n-order-monitoring-alerts (Portfolio)

Production-grade order monitoring in n8n: SLA checks by status, anti-spam cooldown, deterministic event log, centralized error handling.

## What it does

- Accepts webhook `POST /order-status` with payload `{ order_id, status, updated_at }`
- Normalizes input and computes `age_minutes`
- Decides whether an alert is required (SLA):
  - `NEW` too long → `stalled_new` (**warn**)
  - `PAYMENT_PENDING` too long → `stalled_payment` (**critical**)
- Builds `incident_key = order_id + ":" + alert_type`
- Reads incident state from `incidents` and applies cooldown (anti-spam)
- Writes to Google Sheets:
  - `events` — append-only event log
  - `incidents` — current incident state (upsert by `incident_key`)
  - `error_log` — centralized error workflow

## Workflows

- `workflows/order-monitoring.sanitized.json` — main flow (webhook → rules → cooldown → sheets)
- `workflows/error-handling.sanitized.json` — centralized error handling (logs to `error_log`)

## Google Sheets schema

Create a spreadsheet with 3 tabs:

### `events`
Columns:
- `event_id`
- `ts`
- `incident_key`
- `order_id`
- `alert_type`
- `severity`
- `status`
- `details_json`

### `incidents`
Columns:
- `incident_key`
- `order_id`
- `alert_type`
- `severity`
- `status`
- `first_seen_ts`
- `last_seen_ts`
- `last_alert_ts`
- `last_event_id`
- `details_json`

### `error_log`
Columns:
- `ts`
- `workflow`
- `execution_id`
- `node`
- `error_message`
- `stack`
- `payload_json`

## How to run

1) Import both workflows into n8n.
2) Configure Google Sheets credentials and select your spreadsheet/tabs.
3) Send a test request:

```json
{
  "order_id": "ORD-2001",
  "status": "NEW",
  "updated_at": "2026-01-10T20:00:00.000Z"
}
