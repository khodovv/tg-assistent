# Telegram Assistant for n8n (OpenAI + Google Sheets + Google Calendar)

## Артефакты
- `WF_TG_INTAKE_ROUTER.json` — входной роутер Telegram (text/voice/callback), OpenAI intent parser, маршрутизация в handlers.
- `WF_TG_DOMAIN_HANDLERS.json` — отдельные workflow-хендлеры: идеи, список идей, создание/отмена/перенос событий, `/today`.
- `WF_REMINDER_DISPATCHER_AND_DIGEST.json` — Cron напоминаний (каждые 2 мин) + утренний дайджест (08:30).

## Краткая схема логики
1. **WF_TG_INTAKE_ROUTER**
   - Telegram Trigger принимает `message.text`, `message.voice`, `callback_query`.
   - Voice: Telegram file download → OpenAI Transcribe.
   - OpenAI Intent Parser (строго JSON).
   - Switch по `intent` → Execute Workflow нужного обработчика.
   - Логирование в лист `Logs`.
2. **WF_TG_DOMAIN_HANDLERS**
   - `WF_IDEA_ADD_HANDLER`: append в `Ideas` + Telegram confirm.
   - `WF_IDEAS_LIST_HANDLER`: читает `Ideas`, фильтрует `status=new`, отправляет ссылку + топ-10.
   - `WF_EVENT_CREATE_HANDLER`: create event в GCal + `ASSISTANT_UID=<uuid>` в description + запись reminder в `Reminders` + last_created_event в Data Store.
   - `WF_EVENT_CANCEL_HANDLER`: поддержка `отмени её` через Data Store (`last_created_event`).
   - `WF_EVENT_RESCHEDULE_HANDLER`: update last event на новое время.
   - `WF_TODAY_HANDLER`: события на сегодня в TZ Europe/Amsterdam.
3. **WF_REMINDER_DISPATCHER**
   - Каждые 2 минуты читает `Reminders`.
   - Для `status=pending` и `remind_at_iso <= now` отправляет Telegram.
   - Обновляет строку: `sent`/`error`.
4. **WF_MORNING_DIGEST** (опционально)
   - В 08:30 отправляет список событий на сегодня в Telegram.

---

## Настройка credentials в n8n
Создайте credentials:
1. **Telegram Bot API**
   - Название: `Telegram Bot`
   - Токен бота (`TELEGRAM_BOT_TOKEN`)
2. **OpenAI API**
   - Название: `OpenAI`
   - API key (`OPENAI_API_KEY`)
3. **Google OAuth2** (Sheets + Calendar)
   - Название: `Google`
   - Доступ к Google Sheets API и Google Calendar API

> В JSON используются ссылки на env-переменные credential ID (`TELEGRAM_BOT_CREDENTIAL_ID`, `OPENAI_CREDENTIAL_ID`, `GOOGLE_CREDENTIAL_ID`). После импорта можно:
> - либо проставить реальные credential вручную в каждом node,
> - либо использовать ваши internal conventions с env substitutions.

## Переменные окружения
Задайте в n8n (Environment Variables):

```bash
TELEGRAM_BOT_TOKEN=...
OPENAI_API_KEY=...
SPREADSHEET_ID=...
SPREADSHEET_URL=https://docs.google.com/spreadsheets/d/<SPREADSHEET_ID>
CALENDAR_ID=primary
TIMEZONE=Europe/Amsterdam

# IDs credential objects (если используете env substitution)
TELEGRAM_BOT_CREDENTIAL_ID=...
OPENAI_CREDENTIAL_ID=...
GOOGLE_CREDENTIAL_ID=...

# optional
DEFAULT_DIGEST_CHAT_ID=...
```

## Google Sheets: структура листов (строго)
Создайте Spreadsheet: **TG_Assistant_DB** с листами и колонками:

### Ideas
`idea_id, created_at_iso, created_at_local, user_id, username, source_type, category, title, details, tags, status, used_at_iso, link, raw_text, tg_chat_id, tg_message_id`

### Reminders
`reminder_id, created_at_iso, user_id, username, event_id, assistant_uid, remind_at_iso, remind_at_local, remind_offset_min, message, status, sent_at_iso, canceled_at_iso, error, tg_chat_id`

### Settings
`timezone, default_remind_offset_min, calendar_id, ideas_sheet_name, reminders_sheet_name, logs_sheet_name, morning_digest_time`

### Logs
`ts_iso, level, user_id, username, action, intent, confidence, text_in, result_json, error`

## Data Store
Создайте Data Store `tg_assistant_store`.
- ключ `last_created_event:<user_id>`
- value JSON: `{"event_id":"...","assistant_uid":"...","start_iso":"...","title":"..."}`

## Импорт workflows
1. n8n → **Workflows** → **Import from File**.
2. Импортируйте:
   - `WF_TG_INTAKE_ROUTER.json`
   - `WF_TG_DOMAIN_HANDLERS.json`
   - `WF_REMINDER_DISPATCHER_AND_DIGEST.json`
3. В каждом workflow проверьте credentials и параметры.
4. Активируйте workflows.

## OpenAI Intent Parser (system prompt)
Используется внутри node `OpenAI Intent Parser`:
- intents: `idea_add, ideas_list, event_create, event_cancel, event_reschedule, today_digest, help, unknown`
- поля: `idea`, `event`, `cancel`, `reschedule`
- строгий JSON-only output
- timezone: Europe/Amsterdam
- обработка относительных дат

## UX сообщений
- Короткие ответы с маркерами `✅/🕒/🔔`.
- Для неоднозначных отмен/переносов добавьте inline-кнопки callback data:
  - `cancel_event:<event_id>`
  - `reschedule_event:<event_id>`
  - `choose_event:<event_id>`

## Приёмочные тесты
1. **Создание события**
   - Ввод: `Завтра встреча в 14:00, напомни за час`
   - Ожидание: событие в GCal создано; в `description` есть `ASSISTANT_UID=...`; в `Reminders` строка `pending` с `remind_at=13:00`; в Telegram подтверждение.
2. **Отправка напоминания Cron**
   - При достижении `remind_at_iso` workflow `WF_REMINDER_DISPATCHER` отправляет сообщение в Telegram и обновляет `status=sent`.
3. **Добавление идеи**
   - Ввод: `идея для рилса: бэкстейдж раскроя + 3 факта о ткани`
   - Ожидание: строка в `Ideas` с `category=reels`, `status=new`.
4. **Команда `/ideas`**
   - Ожидание: ссылка на таблицу + 10 последних идей со статусом `new`.
5. **Отмена последнего события**
   - Ввод: `отмени её` сразу после создания
   - Ожидание: удаление события из GCal, связанные reminders переводятся в `canceled` (добавьте update-step в cancel handler под ваш режим lookup строк).

## Важно по донастройке
- В `WF_EVENT_CANCEL_HANDLER` и `WF_EVENT_RESCHEDULE_HANDLER` уже реализован быстрый путь через `last_created_event`.
- Для полного сценария поиска по ключевым словам и выбора из нескольких событий добавьте ветку:
  - GCal `getAll` в диапазоне `date_hint ±1 day`
  - фильтр по `keywords/time_hint`
  - если >1 результата, Telegram inline keyboard.
