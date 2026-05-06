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

## Hard prerequisites before wiring any OAuth tool

**Persistence migration first.** OAuth refresh tokens live in the SQLite DB. If `.wa-state.persistence` is `null` or `"ephemeral"`, the tokens get wiped on every redeploy and the user has to revisit `/auth/google` after every code change. Route to `wa-persistence` and pick option ב (Starter + Disk) before continuing — otherwise the user will get an annoying re-auth loop during active dev.

**Audience locked to whitelist.** If `spec.audience.mode` is still `"open"` after deploy, anyone who has the bot's WhatsApp number can ask the bot to read the user's calendar/inbox. Flip to `whitelist` (with the user's phone *and* their LID, since LID-mode is now common — see `wa-build` LID notes) before exposing any personal-data tool.

Confirm both with the user in Hebrew before proceeding:

> לפני שמחברים את {tool_name}, שני דברים חשובים:
>
> 1. **איפה הבוט שומר נתונים?** עכשיו הוא ב-{persistence}. אם נשאר ככה, ההרשאה ל-Google תיעלם בכל פעם שנעדכן את הקוד — תצטרך להתחבר מחדש בדפדפן. כדאי לעבור לדיסק קבוע ($7.25/חודש) קודם.
>
> 2. **מי יכול לדבר עם הבוט?** עכשיו זה {audience.mode}. אם זה פתוח, כל מי שיש לו את המספר יוכל לבקש מהבוט לקרוא את היומן שלך. נעבור לרשימה לבנה?
>
> נסדר את שני אלו ואז נחבר את {tool_name}, או להמשיך כמו שזה?

## Available tools

### `calendar` — Google Calendar

Lets the bot read availability and create/edit/cancel events.

#### User-side prep (Google Cloud Console — ~5 min)

Walk the user through these steps in Hebrew, one at a time. They need to be on https://console.cloud.google.com logged in with the Google account whose calendar the bot will access.

1. **Create a project.** Top bar → project picker → New Project → name it (e.g. "jackbot") → Create. Wait ~10s, select it.
2. **Enable Calendar API.** Top search bar → "Calendar API" → Google Calendar API → Enable.
3. **OAuth consent screen.** Sidebar → APIs & Services → OAuth consent screen → External → Create. Fill App name, support email, dev contact email. Save and Continue through Scopes (default), Test Users (add the user's own email). Save and Continue → Back to Dashboard.
4. **Create OAuth client.** Sidebar → APIs & Services → Credentials → + Create Credentials → OAuth client ID → Web application. Authorized redirect URIs → add **exactly**: `{RENDER_URL}/auth/google/callback`. Click Create.
5. **Copy `client_id` and `client_secret`** from the dialog. They'll go into env vars next.

Ask the user to paste both into the chat in this format:
```
client_id: <paste>
client_secret: <paste>
```

Then save them silently:
- Update local `bot/.env` with `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI={RENDER_URL}/auth/google/callback`, `OWNER_USER_ID=<short_slug>`
- Push the same four vars to Render via `PUT /v1/services/{id}/env-vars/<key>`

#### Code generation

Generate three files, modify two more.

**1. `bot/database.py`** — add the `oauth_tokens` table inside `init_db()`, plus `save_oauth_tokens()` / `get_oauth_tokens()` helpers:

```python
# Inside init_db():
c.execute("""
    CREATE TABLE IF NOT EXISTS oauth_tokens (
      provider TEXT NOT NULL,
      user_id TEXT NOT NULL,
      access_token TEXT,
      refresh_token TEXT,
      token_expiry INTEGER,
      scopes TEXT,
      updated_at INTEGER NOT NULL,
      PRIMARY KEY (provider, user_id)
    )
""")

def save_oauth_tokens(provider, user_id, access_token, refresh_token, expiry, scopes):
    with _conn() as c:
        c.execute("""
            INSERT INTO oauth_tokens (provider, user_id, access_token, refresh_token, token_expiry, scopes, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT (provider, user_id) DO UPDATE SET
              access_token=excluded.access_token,
              refresh_token=COALESCE(excluded.refresh_token, oauth_tokens.refresh_token),
              token_expiry=excluded.token_expiry,
              scopes=excluded.scopes,
              updated_at=excluded.updated_at
        """, (provider, user_id, access_token, refresh_token, expiry, scopes, int(time.time())))

def get_oauth_tokens(provider, user_id):
    with _conn() as c:
        row = c.execute(
            "SELECT access_token, refresh_token, token_expiry, scopes FROM oauth_tokens WHERE provider=? AND user_id=?",
            (provider, user_id),
        ).fetchone()
    if not row:
        return None
    return {"access_token": row[0], "refresh_token": row[1], "expiry": row[2], "scopes": row[3]}
```

**2. `bot/google_oauth.py`** — new module. **CRITICAL: disable PKCE.**

```python
import os
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import Flow
from database import get_oauth_tokens, save_oauth_tokens

CLIENT_ID = os.environ["GOOGLE_CLIENT_ID"]
CLIENT_SECRET = os.environ["GOOGLE_CLIENT_SECRET"]
REDIRECT_URI = os.environ["GOOGLE_REDIRECT_URI"]
OWNER_USER_ID = os.environ.get("OWNER_USER_ID", "owner")
SCOPES = [
    "openid",
    "https://www.googleapis.com/auth/userinfo.email",
    "https://www.googleapis.com/auth/calendar",
]
CLIENT_CONFIG = {
    "web": {
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
        "token_uri": "https://oauth2.googleapis.com/token",
        "redirect_uris": [REDIRECT_URI],
    }
}

def _make_flow() -> Flow:
    flow = Flow.from_client_config(CLIENT_CONFIG, scopes=SCOPES, redirect_uri=REDIRECT_URI)
    # CRITICAL: confidential clients (server with client_secret) do NOT need PKCE.
    # Leaving it enabled causes "(invalid_grant) Missing code verifier" on callback,
    # because /auth/google and /auth/google/callback are stateless handlers — the
    # auto-generated code_verifier from the authorize step is gone by callback time.
    flow.autogenerate_code_verifier = False
    return flow

def build_authorize_url(state: str = "default") -> str:
    flow = _make_flow()
    auth_url, _ = flow.authorization_url(
        access_type="offline",  # required to get a refresh_token
        include_granted_scopes="true",
        prompt="consent",  # forces issuance of refresh_token even on re-auth
        state=state,
    )
    return auth_url

def handle_callback(code: str) -> Credentials:
    flow = _make_flow()
    flow.fetch_token(code=code)
    creds = flow.credentials
    save_oauth_tokens(
        provider="google",
        user_id=OWNER_USER_ID,
        access_token=creds.token,
        refresh_token=creds.refresh_token,
        expiry=int(creds.expiry.timestamp()) if creds.expiry else None,
        scopes=" ".join(creds.scopes or []),
    )
    return creds

def get_user_credentials(user_id: str | None = None) -> Credentials | None:
    user_id = user_id or OWNER_USER_ID
    row = get_oauth_tokens("google", user_id)
    if not row:
        return None
    creds = Credentials(
        token=row["access_token"],
        refresh_token=row["refresh_token"],
        token_uri="https://oauth2.googleapis.com/token",
        client_id=CLIENT_ID,
        client_secret=CLIENT_SECRET,
        scopes=(row["scopes"] or "").split(),
    )
    if creds.expired and creds.refresh_token:
        creds.refresh(Request())
        save_oauth_tokens(
            provider="google",
            user_id=user_id,
            access_token=creds.token,
            refresh_token=creds.refresh_token,
            expiry=int(creds.expiry.timestamp()) if creds.expiry else None,
            scopes=" ".join(creds.scopes or []),
        )
    return creds
```

**3. `bot/main.py`** — add the OAuth routes. Import once near the top:
```python
from fastapi.responses import PlainTextResponse, RedirectResponse
from google_oauth import build_authorize_url, handle_callback
```

Then below the existing routes:
```python
@app.get("/auth/google")
async def auth_google():
    return RedirectResponse(build_authorize_url())

@app.get("/auth/google/callback")
async def auth_google_callback(code: str | None = None, error: str | None = None):
    if error:
        return PlainTextResponse(f"OAuth error: {error}", status_code=400)
    if not code:
        return PlainTextResponse("Missing code", status_code=400)
    try:
        handle_callback(code)
    except Exception as e:
        logger.exception("oauth callback failed")
        return PlainTextResponse(f"OAuth callback error: {e}", status_code=500)
    return PlainTextResponse("Connected ✅. Tokens saved. You can close this tab.")
```

**4. `bot/tools/calendar_tool.py`** — the tool module. **Name it `calendar_tool.py`, NOT `calendar.py`** — Python has a stdlib `calendar` module and the auto-import in `tools/__init__.py` will shadow it, breaking other imports in subtle ways.

```python
from datetime import datetime, timezone
from googleapiclient.discovery import build
from google_oauth import get_user_credentials
from .registry import register

def _service():
    creds = get_user_credentials()
    if not creds:
        return None
    return build("calendar", "v3", credentials=creds, cache_discovery=False)

_NOT_CONNECTED = (
    "אני לא מחובר ליומן עדיין. בקר בקישור הזה בדפדפן כדי לחבר: "
    "{RENDER_URL}/auth/google"
)

async def list_upcoming_events(chat_id: str, max_results: int = 10) -> str:
    svc = _service()
    if not svc:
        return _NOT_CONNECTED
    now = datetime.now(timezone.utc).isoformat()
    result = svc.events().list(
        calendarId="primary", timeMin=now,
        maxResults=max(1, min(int(max_results), 25)),
        singleEvents=True, orderBy="startTime",
    ).execute()
    events = result.get("items", [])
    if not events:
        return "אין אירועים קרובים ביומן."
    lines = []
    for e in events:
        when = e["start"].get("dateTime") or e["start"].get("date")
        title = e.get("summary", "(ללא כותרת)")
        lines.append(f"• {when}: {title}  [id={e.get('id','')}]")
    return "\n".join(lines)

# create_event and cancel_event — see wa-build's existing tool template.

register(
    name="list_upcoming_events",
    description=(
        "List the user's upcoming Google Calendar events from the primary calendar. "
        "Use when the user asks what's on their calendar, what's coming up, when they're free."
    ),
    input_schema={
        "type": "object",
        "properties": {
            "max_results": {"type": "integer", "description": "Max events to return (1-25), default 10"},
        },
        "required": [],
    },
    handler=list_upcoming_events,
)
# (also register create_event, cancel_event)
```

**5. `bot/requirements.txt`** — add the three Google libs:
```
google-auth>=2.32.0
google-auth-oauthlib>=1.2.0
google-api-python-client>=2.140.0
```

#### Deploy and consent

After pushing the code:
1. Trigger Render deploy via API (`POST /v1/services/{id}/deploys` with `{}`) — the GitHub-webhook auto-deploy is slow (~30+s after push); manual trigger is faster.
2. Wait for `live` status, then wait an extra ~30s for cold-start to actually serve (`/healthz` returns 200).
3. Verify `/auth/google` returns HTTP 307 with a Location header that contains `client_id`, `access_type=offline`, `prompt=consent` and **does NOT contain `code_challenge`** (that confirms PKCE is off).
4. Tell the user (in Hebrew) to open `{RENDER_URL}/auth/google` in their browser:
   - Pick their Google account
   - Click **Advanced → Go to {bot_name} (unsafe)** (app is in Testing mode)
   - Continue through scopes, click Allow
   - Land on the "Connected ✅. Tokens saved." page
5. They tell you done. Test from WhatsApp: "מה יש לי ביומן השבוע?"

#### Verifying tool fired

Watch Render logs for the `list_upcoming_events` invocation. The webhook closes 200 OK ~3-5s after inbound when everything works. If `_NOT_CONNECTED` reply came back, tokens are missing — the user didn't complete the consent flow successfully. If `googleapiclient` errors appear in logs, scope or credentials issue.

⚠️ **OAuth refresh token storage** must be on persistent disk (see prerequisites). Without it, a redeploy clears the DB and the user has to revisit `/auth/google` again.

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
