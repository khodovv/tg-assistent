# n8n Telegram Personal Assistant (OpenAI + Google Sheets + Google Calendar)

## 1) Google Sheets template (exact tabs + columns)
Spreadsheet: `TG_Assistant_DB`

### Config
`key, value, updated_at_iso`

Required keys in `Config`:
- `timezone` = `Europe/Amsterdam`
- `default_remind_policy_minutes` = `[60,15]`
- `quiet_hours_start` = `23`
- `quiet_hours_end` = `8`
- `calendar_id` = `primary`
- `daily_digest_morning` = `09:00`
- `daily_digest_evening` = `21:30`

### Tasks
`task_id, created_at_iso, updated_at_iso, user_id, username, title, details, status, priority, due_at_iso, timezone, remind_policy_minutes, next_remind_at_iso, last_reminded_at_iso, calendar_event_id, calendar_sync_token, source, dedupe_key, tg_chat_id, tg_message_id, snooze_until_iso, done_at_iso, canceled_at_iso`

### Ideas
`idea_id, created_at_iso, user_id, username, category, title, details, tags_json, status, source, dedupe_key, tg_chat_id, tg_message_id`

### Logs
`log_id, ts_iso, level, workflow, user_id, username, action, intent, source, input_text, payload_json, error`

---

## 2) Workflows JSON (for n8n 2.x import)
- `TG_Inbound_Router.json`
- `Ideas_Create.json`
- `Tasks_CreateOrUpdate.json`
- `Reminder_Worker.json`
- `Calendar_Sync_5min.json`

### Import order
1. `Ideas_Create`
2. `Tasks_CreateOrUpdate`
3. `Reminder_Worker`
4. `Calendar_Sync_5min`
5. `TG_Inbound_Router`

---

## 3) Credentials and env

### Credentials
- Telegram Bot API
- OpenAI API
- Google OAuth2 (Sheets + Calendar)

### Env
```bash
TELEGRAM_BOT_TOKEN=...
SPREADSHEET_ID=...
CALENDAR_ID=primary
DEFAULT_TIMEZONE=Europe/Amsterdam
QUIET_HOURS_START=23
QUIET_HOURS_END=8
DEFAULT_CHAT_ID=

TELEGRAM_BOT_CREDENTIAL_ID=...
OPENAI_CREDENTIAL_ID=...
GOOGLE_CREDENTIAL_ID=...
```

Data Store name: `tg_assistant_store`
- key: `last_calendar_sync` (JSON: `{ "ts": "...ISO..." }`)

---

## 4) Logic scheme (short)
1. **TG_Inbound_Router**: Telegram trigger → voice transcribe → OpenAI strict JSON intent → route.
2. **Ideas_Create**: append row to `Ideas` + Telegram confirmation.
3. **Tasks_CreateOrUpdate**: upsert to `Tasks` by `dedupe_key`; optional Calendar create; compute `next_remind_at_iso` using policy + quiet hours.
4. **Reminder_Worker**: cron each minute → due tasks → Telegram reminder with callback buttons (`task_done:<id>`, `task_snooze:<id>:10`, `task_cancel:<id>`) → update task + logs.
5. **Calendar_Sync_5min**: cron each 5 min → fetch changed events since `last_calendar_sync` → upsert by `calendar_event_id` (via `dedupe_key=calendar:<eventId>`) → set `last_calendar_sync`.

---

## 5) Callback data spec (strict)
- done: `task_done:<task_id>`
- snooze: `task_snooze:<task_id>:<minutes>`
- cancel: `task_cancel:<task_id>`

---

## 6) Idempotency rules
- Tasks upsert uses `dedupe_key` (`task:user:title:date` or `calendar:<eventId>`).
- Ideas include `dedupe_key` for duplicate filtering downstream.
- Calendar sync state persisted in Data Store `last_calendar_sync`.

---

## 7) Acceptance tests
1. Create/update task from Telegram text → row appears in `Tasks`, no duplicate on retry (same dedupe key).
2. Voice message: transcription passes through router and reaches Ideas/Tasks flow.
3. Reminder minute cron sends Telegram with 3 callback buttons and updates task reminder fields.
4. Calendar sync creates/updates tasks by `calendar_event_id` and advances `last_calendar_sync`.
5. Errors/warnings are appended to `Logs` with payload and workflow name.

