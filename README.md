# Deletion Monitor

A Telegram bot that monitors Telegram chats for deleted messages using the official Telegram MTProto protocol through Telethon. It is a full async application with PostgreSQL, Redis, Alembic, FastAPI account linking, an admin panel, and four languages: Uzbek, Russian, English, and Chinese.

The default account-linking flow runs inside the bot chat: phone number, Telegram login code, and the 2FA password if enabled. A separate HTTPS QR page is still available as an alternative. Credentials are never written to the database or logs, and the bot tries to delete the messages that contain them.

## What this bot can and cannot do

### What it does

- Connects a user's own Telegram account through an in-chat phone/code/2FA flow, with an optional HTTPS QR page.
- Monitors new messages in groups, supergroups, and channels where that account is a member.
- Stores a short-lived local cache of message metadata (sender, chat, text preview, media type).
- Correlates `MessageDeleted` events with cached messages and sends a notification to the user.
- Lets each user pause monitoring, disable notifications, change retention, clear cache, and view history.
- Provides an admin panel with users, bans, error logs, and system health.

### Honest limitations

- **Secret chats are not supported.** Telegram's client protocol does not expose them to third-party clients in a way this bot can monitor.
- **Deletion events are not guaranteed.** Telegram does not always deliver `MessageDeleted` updates. Some deletions can be missed.
- **Deleted content is only shown if it was cached before deletion.** This bot cannot recover content it never received.
- **Only chats where the linked account is a member are visible.** Private conversations of other people are not accessible.
- **"Delete for me only" is invisible to other devices.** If a sender deletes a message only for themselves, other Telegram clients, including this bot, do not receive a deletion event.
- **Message edits are not deletion events.** Editing a message is not reported as a deletion by Telegram.
- **Telegram limits and anti-spam measures apply.** Flood waits, session invalidation, and account restrictions are handled conservatively but cannot be bypassed.
- **The linked account must stay logged in.** If Telegram invalidates the session, the bot marks the account inactive and asks the user to reconnect.
- **Account linking requires one application process.** Link flows are kept in memory. Run exactly one web/bot instance, or use sticky sessions and shared state if you later scale out. This repository is intentionally configured for one instance.

Use this bot only with your own Telegram account and in compliance with Telegram's Terms of Service and local privacy laws. Users are responsible for the chats they monitor and for informing participants where required.

## Features

- Official Telegram QR account linking page and an in-chat phone/code/2FA login flow; optional local `StringSession` import for automation.
- Fernet encryption for stored Telegram session strings.
- Session strings and passwords are never written to logs; link tokens are short-lived and revoked after use.
- Telegram account is disconnected and its stored secret is removed on user request or admin ban.
- Per-user settings: notifications, content storage, monitoring pause, cache retention (1/7/30 days).
- Automatic message-cache retention cleanup and old error-log pruning.
- Redis-backed distributed lock per account with an in-memory fallback; Redis is strongly recommended in production.
- FastAPI health endpoint, secure link page, optional Telegram webhook, and polling mode.
- Alembic migrations and Docker/Render deployment files.
- Four-language UI with English fallback.

## Architecture

```
Telegram Bot API <-> aiogram dispatcher <-> handlers / admin panel
Telegram MTProto  <-> Telethon monitor -> message cache -> deletion events -> notifier
FastAPI           <-> secure link page -> LinkFlowManager -> encrypted session storage
PostgreSQL        <-> SQLAlchemy async models
Redis             <-> locks, link tokens, ephemeral state
```

Key runtime pieces:

- `api/app.py`: FastAPI lifespan, bot/dispatcher startup, monitor startup, cleanup tasks, polling or webhook mode.
- `services/monitor.py`: Telethon lifecycle, dialog sync, message caching, deletion handling, notifications.
- `api/link_manager.py`: QR login, 2FA password handling, StringSession import, encrypted persistence.
- `middlewares/user_context.py`: onboarding, ban checks, admin injection, translation binding.
- `services/cleanup.py`: retention and error-log cleanup.

## Requirements

- Python 3.12 or 3.13 recommended. Python 3.14 may work, but some native dependencies such as `asyncpg` may not have wheels yet.
- PostgreSQL 14+.
- Redis 6+ (optional for local experimentation; required for reliable production operation).
- A Telegram bot token from [@BotFather](https://t.me/BotFather).
- Telegram API ID and API hash from [my.telegram.org/apps](https://my.telegram.org/apps).

## Quick start with Docker

1. Copy the environment template:

   ```powershell
   Copy-Item .env.example .env
   ```

2. Fill the required values in `.env`:

   - `BOT_TOKEN`
   - `ADMIN_IDS`
   - `TELEGRAM_API_ID`
   - `TELEGRAM_API_HASH`
   - `SESSION_ENCRYPTION_KEY`
   - `PUBLIC_BASE_URL`

   Generate the encryption key once:

   ```powershell
   py -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
   ```

3. Start PostgreSQL, Redis, and the bot:

   ```powershell
   docker compose up --build -d
   docker compose logs -f bot
   ```

4. Open `http://localhost:8080/health`. It should return a healthy JSON response.

5. Send `/start` to the bot, choose a language, then use **Link account**.

The compose file overrides `DATABASE_URL` and `REDIS_URL` with the internal container addresses. Other values come from `.env`.

## Local development

Create and activate a virtual environment:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements-dev.txt
```

If you are using Python 3.14 and `asyncpg` fails to build, use Python 3.12/3.13 for local development, use Docker, or use SQLite for non-production UI testing:

```env
DATABASE_URL=sqlite+aiosqlite:///./local.db
REDIS_URL=redis://127.0.0.1:6379/0
```

Create the database schema. Production should use Alembic:

```powershell
alembic upgrade head
```

For a quick local setup, the application also runs `Base.metadata.create_all` at startup. The script below does the same thing explicitly:

```powershell
py scripts/init_db.py
```

Run the application:

```powershell
py main.py
```

The app serves the bot and the web page in one process. Do not run multiple workers.

## Telegram setup

### 1. Create the bot

Create a bot with @BotFather and copy the token into `BOT_TOKEN`. Set `BOT_USERNAME` to the bot username without `@` so the link page can show a "back to bot" button.

### 2. Create Telegram API credentials

Go to [my.telegram.org/apps](https://my.telegram.org/apps), create an application, and copy its API ID and API hash. These are used by Telethon, not by the bot token.

### 3. Set admin IDs

`ADMIN_IDS` is a comma-separated list of numeric Telegram user IDs. Those users see the admin panel button and can use `/admin`.

## Account linking and security

The bot supports two linking paths. The in-chat flow is the default; the HTTPS QR page remains available for users who prefer scanning.

### In-chat linking (default)

1. The user presses **Link account** in the bot.
2. The bot asks for the phone number (a contact button is provided).
3. Telethon sends the Telegram login code. The user types it in the bot chat.
4. If two-step verification is enabled, the user types the 2FA password in the bot chat.
5. The bot deletes the messages containing the phone number, code, and password where Telegram permits.
6. The session string is encrypted with `SESSION_ENCRYPTION_KEY` and stored in the database, then monitoring starts.

The code and password are only held in process memory during the login flow. They are never written to the database or logs. Because they are typed in a chat, treat the chat as sensitive: enable 2FA, use a private chat with the bot, and clear the history after linking.

### HTTPS QR linking (optional)

1. The bot creates a short-lived token and sends a button to the secure HTTPS page.
2. The page starts an official Telegram QR login. The user scans it from Telegram: Settings, Devices, Link Desktop Device.
3. If two-step verification is enabled, the user enters the password only on the HTTPS page. The password is passed to Telethon and is not stored.
4. The session string is encrypted with `SESSION_ENCRYPTION_KEY` and stored in the database.
5. The temporary token is revoked and monitoring starts for the new account.

Security rules enforced by the application:

- Login codes and 2FA passwords are not stored in the database or logs.
- Messages with codes and passwords are deleted after use where Telegram permits.
- Link tokens expire after `LINK_TOKEN_TTL_MINUTES` and are revoked on completion.
- Link pages send `Cache-Control: no-store`, no-referrer, frame denial, and a restrictive CSP.
- Session strings are stripped from API errors and error logs.
- Sessions are encrypted at rest. Changing or losing `SESSION_ENCRYPTION_KEY` makes existing sessions unreadable.
- Disconnecting an account removes the encrypted session, clears cached message content, and stops monitoring.
- Banning a user disconnects their account and stops monitoring.

### Optional StringSession import

If you need to import an existing Telethon session, create it locally:

```powershell
$env:TELEGRAM_API_ID="123456"
$env:TELEGRAM_API_HASH="your_api_hash"
py scripts/create_session.py
```

The script prints the session string to standard output and warnings to standard error. Treat this string exactly like a password. Paste it only into the bot's HTTPS linking page, never into a chat. Anyone who obtains it can use the Telegram account.

## Environment variables

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `BOT_TOKEN` | Yes | - | Telegram bot token from BotFather. |
| `BOT_USERNAME` | No | empty | Bot username without `@`, used on the link page. |
| `ADMIN_IDS` | Yes | empty | Comma-separated numeric Telegram user IDs for the admin panel. |
| `DATABASE_URL` | Yes | local Postgres | `postgresql+asyncpg://...` or `sqlite+aiosqlite:///...`. `postgres://` is normalized automatically. |
| `REDIS_URL` | No | `redis://localhost:6379/0` | Redis for locks and link tokens. Falls back to process memory if unavailable. |
| `SESSION_ENCRYPTION_KEY` | Yes | empty | Fernet key used to encrypt Telegram sessions. Keep it stable and secret. |
| `TELEGRAM_API_ID` | Yes for linking | `0` | Telegram MTProto API ID. |
| `TELEGRAM_API_HASH` | Yes for linking | empty | Telegram MTProto API hash. |
| `PUBLIC_BASE_URL` | Yes | `http://localhost:8080` | Public URL of this service, no trailing slash. Must be HTTPS in production. |
| `PORT` | No | `8080` | HTTP port. Render sets this automatically. |
| `USE_WEBHOOK` | No | `false` | `true` to use the Telegram webhook instead of polling. |
| `WEBHOOK_SECRET` | When webhook is on | empty | Secret Telegram sends in `X-Telegram-Bot-Api-Secret-Token`. |
| `WEBHOOK_PATH` | No | `/telegram/webhook` | Webhook route. |
| `DEFAULT_LANGUAGE` | No | `uz` | Fallback language before a user chooses one. |
| `DISPLAY_TIMEZONE` | No | `Asia/Tashkent` | Timezone used in stats and notifications. |
| `MONITORING_ENABLED` | No | `true` | Global monitoring kill switch. |
| `LINK_TOKEN_TTL_MINUTES` | No | `15` | Lifetime of account-linking tokens. |
| `MONITOR_SYNC_INTERVAL_SECONDS` | No | `300` | How often monitored dialogs are resynced. |
| `MONITOR_LOCK_TTL_SECONDS` | No | `90` | Redis lock TTL for an account monitor. |
| `MAX_NOTIFICATION_LENGTH` | No | `700` | Maximum cached text preview shown in a deletion notification. |
| `SESSION_STRING_MAX_LENGTH` | No | `4096` | Maximum accepted StringSession length. |
| `LOG_LEVEL` | No | `INFO` | Python log level. |
| `ENVIRONMENT` | No | `development` | Set to `production` on a deployed service. |

## Database and migrations

The initial migration is in `alembic/versions/0001_initial.py`. Run migrations before starting a production process:

```powershell
alembic upgrade head
```

Create a new migration after changing models:

```powershell
alembic revision --autogenerate -m "describe change"
alembic upgrade head
```

The Docker image runs `alembic upgrade head` before starting the app.

Useful tables:

- `users`, `user_settings`
- `connected_accounts`
- `monitored_chats`, `message_metadata`
- `deletion_events`, `notification_history`
- `admins`, `error_logs`

Message metadata is deleted according to each user's retention setting. Error logs are pruned after 30 days. Deletion event history is kept until the user clears it or the account is removed with event clearing enabled.

## Running checks

```powershell
py -m ruff check .
py scripts/check_locales.py
py -m pytest -q
```

The test suite uses SQLite and an in-memory cache fixture; it does not contact Telegram, PostgreSQL, or Redis.

## Deploying to Render

`render.yaml` defines:

- A Docker web service.
- A Render PostgreSQL database.
- A Render Key Value (Redis-compatible) instance.

Steps:

1. Push this directory to a Git repository that Render can access.
2. In Render, create a new Blueprint and select the repository.
3. Set the secret environment variables when prompted:

   - `BOT_TOKEN`
   - `BOT_USERNAME`
   - `ADMIN_IDS`
   - `TELEGRAM_API_ID`
   - `TELEGRAM_API_HASH`
   - `SESSION_ENCRYPTION_KEY`
   - `PUBLIC_BASE_URL` (for example `https://deletion-monitor.onrender.com`)

4. Apply the Blueprint. The first deploy builds the Docker image and runs Alembic migrations.
5. Confirm that `https://your-service.onrender.com/health` is healthy.
6. Send `/start` to the bot and link an account.

Important deployment notes:

- **Use always-on paid instances.** Free web services spin down when idle, which stops monitoring and QR link flows. `render.yaml` uses paid Starter/Basic plans.
- **Keep exactly one instance.** The Blueprint sets the minimum and maximum instance count to one. Do not scale horizontally without replacing in-memory link flows and reviewing monitor locks.
- **Set `PUBLIC_BASE_URL` to the public HTTPS URL.** Production startup rejects a non-HTTPS base URL.
- **Keep `SESSION_ENCRYPTION_KEY` stable.** Changing it invalidates every stored Telegram session.
- **Polling is the default.** `USE_WEBHOOK=false` works on a single always-on Render web service. If you enable webhooks, set a strong `WEBHOOK_SECRET`.
- **Do not use a free expiring PostgreSQL database** for a long-running monitor. Choose a persistent paid plan.

### Webhook mode

To use webhooks:

```env
USE_WEBHOOK=true
WEBHOOK_SECRET=replace-with-a-long-random-secret
PUBLIC_BASE_URL=https://your-service.example.com
```

The app sets the webhook at startup and removes it on shutdown. The webhook route validates Telegram's secret header. The link page still works independently.

## Operations

- Check health: `GET /health`.
- View logs: `docker compose logs -f bot`, or the Render logs dashboard.
- Admin panel: `/admin` in the bot for IDs in `ADMIN_IDS`.
- Restart after changing `.env` values.
- If Redis is down, the app logs a warning and uses in-memory state. This is acceptable for a single process but loses locks and link tokens on restart.
- If a linked account stops working, the user should disconnect it and link again. Admin bans also disconnect accounts.
- If Telegram floods the account, the monitor backs off and logs the event. Repeated flood waits may require waiting before reconnecting.

## Troubleshooting

### The bot starts but the link page says linking is not configured

Set `TELEGRAM_API_ID`, `TELEGRAM_API_HASH`, and `SESSION_ENCRYPTION_KEY`, then restart.

### The QR code expires immediately

Telegram QR login tokens are short-lived. Press refresh on the page. If Telegram returns a flood wait, wait before retrying.

### The account links but no deletions arrive

Check all of the following:

- The account is a member of the chat.
- The chat is not a secret chat.
- Monitoring is enabled in settings.
- Notifications are enabled.
- The message was sent after the account was linked and cached.
- Telegram actually delivered a deletion update. Deletion updates are not guaranteed.

### The link page URL is rejected in production

`PUBLIC_BASE_URL` must start with `https://` when `ENVIRONMENT=production`. Use a TLS-terminating proxy or the Render URL.

### Sessions became unreadable after a deploy

`SESSION_ENCRYPTION_KEY` changed. Restore the original key. If it is lost, users must reconnect their accounts.

## Project structure

```
admin/                  Admin filters, handlers, router
alembic/                Alembic environment and initial migration
api/                    FastAPI app, link manager, routes, template
bot/                    aiogram bot and dispatcher factory
config/                 Pydantic settings
database/               SQLAlchemy base, engine, session helpers
handlers/               Bot command and callback handlers
keyboards/              Inline keyboards
localization/           Translator and uz/ru/en/zh locale files
middlewares/            Database, user context, throttling middleware
models/                 SQLAlchemy ORM models
scripts/                Session creation, DB init, locale checks
services/               Accounts, cache, crypto, monitor, notifier, stats, cleanup
tests/                  Offline unit tests
main.py                 Uvicorn entry point
```
