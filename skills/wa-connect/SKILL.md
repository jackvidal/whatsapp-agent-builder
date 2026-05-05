---
name: wa-connect
description: Stage 5. Add tools to the live bot one at a time — Google Calendar, Gmail, persistent reminders, human handoff, group chat, voice transcription. Each tool is wired, deployed, and verified before moving to the next.
---

# wa-connect — Add tools, one at a time

Goal: extend the bot with real-world capabilities. **One tool per session.** Never wire two tools at once — when something breaks the user can't tell which addition caused it.

**Speak Hebrew. Confirm before each integration. Test after each integration.**

## How this skill works

1. Read `spec.tools_wishlist` and `.wa-state.connected_tools`. The diff is what's left to wire.
2. Show the user the menu of available tools, marked with which are already connected.
3. They pick one.
4. You wire it: write the tool file in `bot/tools/`, register it in `tools/__init__.py` (or via auto-import), update `bot/requirements.txt` if needed, push to GitHub. Render auto-deploys.
5. Verify the tool by triggering a test from a real WhatsApp message.
6. Update `.wa-state.connected_tools`.
7. Ask if they want another tool or stop.

## Available tools

### `calendar` — Google Calendar
Lets the bot read availability and create/edit/cancel events.

**OAuth flow.** Walk the user through:
1. Create a Google Cloud project at https://console.cloud.google.com
2. Enable Calendar API
3. Configure OAuth consent screen (External, scopes: `calendar.events`, `calendar.readonly`)
4. Create OAuth credentials (Web app), add Render URL as redirect URI
5. Save `client_id` + `client_secret` as Render env vars
6. Run a one-time auth flow: visit `{render_url}/auth/google`, get tokens stored in DB

Generate `bot/tools/calendar.py`:
- Functions: `list_upcoming_events`, `create_event`, `cancel_event`
- Stores refresh token in SQLite (`oauth_tokens` table, keyed by chat_id or owner)
- Adds `/auth/google` and `/auth/google/callback` routes to `main.py`

⚠️ Token storage: if user picked SQLite-on-disk persistence (default), refresh tokens survive restarts but **not** Render redeploys (free tier wipes disk). Recommend wa-persistence to migrate to Postgres before relying on this in production.

### `gmail` — Gmail
Read recent emails, summarize threads, send drafts.

Same OAuth setup as calendar but with Gmail scopes (`gmail.readonly`, `gmail.send`, `gmail.compose`). Re-uses the same `oauth_tokens` table.

Generate `bot/tools/gmail.py`:
- Functions: `list_recent_emails`, `summarize_thread`, `send_email`
- Always confirms before `send_email` — quote the draft to the user via WhatsApp and wait for "אישור" before actually sending.

### `reminders_persistent` — Reminders that survive restarts
Upgrades the in-memory reminders from `wa-build` to APScheduler with SQLite jobstore.

Generate `bot/tools/reminders_persistent.py` and add `apscheduler==3.10.*` to `requirements.txt`. Replace the in-memory `_pending` list with a `BackgroundScheduler` configured with `SQLAlchemyJobStore('sqlite:///./bot.db')`.

Tools exposed:
- `schedule_reminder(in_minutes_or_datetime, message, recurring=false)`
- `list_my_reminders()`
- `cancel_reminder(id)`

### `human_handoff` — Escalate to a person
When triggered, the bot sends a message to `spec.handoff.owner_number` (the user's number) with a summary of what the customer wants, and tells the customer "מישהו יחזור אליך עוד מעט."

Generate `bot/tools/human_handoff.py`. Auto-fires when:
1. The LLM detects intent matching `spec.handoff.trigger_phrases`, OR
2. The bot fails 3 times in a row to answer (sentiment fallback)

The handoff message to the owner includes:
- The customer's number
- Last 5 messages of context
- A "click to reply" link: `https://wa.me/{customer_number}`

### `groups` — Group chat support
By default `main.py` ignores group messages. This tool flips a config flag `GROUPS_ENABLED=true` and changes the routing in `main.py` to:
- Treat group `chat_id` as the conversation ID
- Only respond when the bot is mentioned (@<bot_phone>) OR the message starts with the configured prefix (default `/`)

Ask the user:
> איך הבוט יידע שמדברים אליו בקבוצה?
> 1. כשמזכירים אותו (@)
> 2. כשמתחילים הודעה ב-`/`
> 3. שניהם

### `voice` — Voice note transcription
Wasender delivers voice notes encrypted. Wire `POST /api/decrypt-media` to fetch a temp public URL, then transcribe with OpenAI Whisper or Anthropic's audio handling.

Generate `bot/tools/voice.py`:
- In `main.py`, when `data.messages.message.audioMessage` is present (instead of `conversation`), call `decrypt_media(audio_block)` → temp URL → transcribe → feed transcribed text into `handle_message` as if it were a text message.
- Add `OPENAI_API_KEY` env var if using Whisper, or use Anthropic Sonnet's audio understanding directly.

## The flow per tool

For whichever tool the user picks:

### Step A — Confirm scope
Tell the user (in Hebrew) what this tool will let the bot do, what permissions it requires, and what could go wrong. Example for calendar:

> 📅 **חיבור יומן Google**
>
> אחרי החיבור, הבוט יוכל:
> - לראות את הפגישות הקרובות שלך
> - לקבוע פגישות חדשות
> - לבטל / לדחות פגישות
>
> מה צריך ממך:
> - חשבון Google
> - אישור הרשאות (תהליך OAuth קצר)
>
> לוקח לי בסך הכל בערך 5 דקות. נתחיל?

### Step B — External setup (OAuth, API keys)
Walk through any one-time external setup. **Never paste credentials into the chat output.** Save to `.env` and Render env vars.

### Step C — Generate the tool file
Write `bot/tools/<tool_name>.py`. Each tool file:
1. Imports `register` from `tools.registry`
2. Defines async handler functions with signature `async def fn(chat_id: str, **kwargs) -> str`
3. Calls `register(...)` at module level for each function exposed to the LLM

### Step D — Make sure the tool gets imported
Either auto-import all `bot/tools/*.py` in `tools/__init__.py`:
```python
import importlib, pkgutil
for _, modname, _ in pkgutil.iter_modules(__path__):
    if modname not in ("registry",):
        importlib.import_module(f"{__name__}.{modname}")
```

Or explicitly import in `main.py`. Auto-import is simpler for non-technical users — fewer steps to forget.

### Step E — Update requirements.txt if needed
Add new pip packages. Pin majors only.

### Step F — Push to GitHub, wait for Render redeploy
```bash
cd bot
git add tools/<name>.py requirements.txt main.py
git commit -m "add tool: <name>"
git push
```

Render auto-deploys. Poll `GET /v1/services/{id}/deploys?limit=1` until `live`. Show Hebrew progress.

### Step G — Verify
Tell the user:
> שלח לבוט הודעה כמו: "{example_trigger}". אם הוא משתמש ביכולת החדשה, סיימנו.

Watch Render logs for the tool invocation. If the tool fires correctly, success. If not, debug:
1. Did the LLM see the tool? (Check `tools` array in the request — log it temporarily)
2. Did the LLM call it? (Check `tool_use` block in the response)
3. Did the handler raise? (Check exception logs)

### Step H — Update state
Append the tool name to `.wa-state.connected_tools`. Update `last_updated`.

### Step I — Continue or stop?

> 🎉 {tool_name} מחובר ועובד.
>
> רוצה להוסיף עוד כלי, או לעצור כאן? תמיד אפשר לחזור (`/wa`).

## Pitfalls

- **OAuth redirect URI mismatch.** Google OAuth requires the exact redirect URI to be registered. After deploying to Render, update Google Cloud Console with the actual Render URL — not the placeholder.
- **Calendar refresh token expires.** If the user revokes access in their Google account, the refresh token dies. The tool should catch the exception, send the user a Hebrew "החיבור פג, צריך לחבר מחדש" message, and disable itself until re-auth.
- **Voice transcription costs.** Each voice note hits an external API (Whisper or Sonnet audio). Mention rough cost (~$0.006/minute Whisper) so the user isn't surprised on the bill.
- **Group join spam.** When the bot is added to a group, it might receive every message. Make sure the @-mention or `/` prefix gating is on by default.
- **Handoff loop.** If the human handoff fires too aggressively (e.g. on the bot's own first sentence containing "נציג"), debug `trigger_phrases` and tighten the matching to whole-word.
