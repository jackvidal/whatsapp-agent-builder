---
name: wa-build
description: Stage 3. Generate the complete FastAPI bot project from spec.json. Writes main.py, agent.py, database.py, tools/whatsapp.py, prompt.py, requirements.txt, and .env. Wasender-aware throughout. Runs the bot locally to verify it works before deploying.
---

# wa-build — Generate the FastAPI bot

Goal: take `spec.json` from the previous stage and produce a complete, runnable FastAPI bot. The user does nothing here except watch and confirm. **You write every file.** No templates copied — generate fresh from spec so the code traces directly to the user's choices.

**Speak Hebrew while you work** (one-line status updates so the user knows you're alive). Don't dump the code into the chat — write it to disk.

## Core architecture

```
bot/
├── main.py              # FastAPI app, /webhook/wasender, /healthz
├── agent.py             # LLM loop, tool calling, history
├── prompt.py            # System prompt built from spec.json
├── database.py          # SQLite conversations + dedupe
├── tools/
│   ├── __init__.py
│   ├── registry.py      # TOOL_REGISTRY — single source of truth
│   ├── whatsapp.py      # Wasender client: send_text, send_image, ...
│   └── reminders.py     # First tool wired in (fast first win)
├── requirements.txt
├── .env.example
├── .env                 # gitignored — populated from project-root .env
├── .python-version      # 3.12.7
├── .gitignore
└── spec.json            # copied from project root
```

Place this `bot/` folder inside the user's project directory.

## Step 1 — Read spec.json

Load `./spec.json`. Extract:
- `name`, `archetype`, `language_mode`, `tone`, `scope`, `audience`, `knowledge_base`, `tools_wishlist`, `handoff`, `llm`

Validate that all required fields exist. If something is missing, tell the user and route back to `wa-characterize`.

## Step 2 — Tell the user what's happening

> בונה עכשיו את הבוט. אכתוב את כל הקבצים, אתקין תלויות, ואריץ אותו על המחשב שלך כדי לוודא שהוא עונה.
>
> ייקח כדקה.

## Step 3 — Generate `bot/main.py`

This is the FastAPI app. Critical Wasender-specific details:

```python
import hmac, json, logging, os
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, HTTPException, status
from dotenv import load_dotenv

from database import init_db, seen_message
from agent import handle_message

load_dotenv()
logger = logging.getLogger("wa-bot")
logging.basicConfig(level=logging.INFO)

WEBHOOK_SECRET = os.environ["WASENDER_WEBHOOK_SECRET"]

@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()
    yield

app = FastAPI(lifespan=lifespan)

def verify_signature(header_value: str | None) -> bool:
    # Wasender sends the literal webhook_secret as X-Webhook-Signature.
    # Not HMAC. Constant-time compare anyway.
    if not header_value:
        return False
    return hmac.compare_digest(header_value, WEBHOOK_SECRET)

@app.post("/webhook/wasender")
async def wasender_webhook(request: Request):
    if not verify_signature(request.headers.get("X-Webhook-Signature")):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "bad signature")

    payload = await request.json()
    event = payload.get("event")

    if event != "messages.received":
        # Ignore everything except inbound text for now (status updates, deliveries, etc.)
        return {"ok": True, "ignored": event}

    data = payload.get("data", {}).get("messages", {})
    key = data.get("key", {})
    msg_id = key.get("id")
    from_me = key.get("fromMe", False)

    if from_me or not msg_id:
        return {"ok": True, "ignored": "self_or_no_id"}

    # Dedupe — Wasender may retry on 5xx
    if seen_message(msg_id):
        return {"ok": True, "ignored": "duplicate"}

    sender_phone = data.get("cleanedSenderPn") or key.get("remoteJid", "").split("@")[0]
    text = data.get("messageBody")

    if not sender_phone or not text:
        return {"ok": True, "ignored": "no_text_or_sender"}

    chat_id = f"+{sender_phone}"

    try:
        await handle_message(chat_id=chat_id, text=text)
    except Exception as e:
        logger.exception("handle_message failed: %s", e)
        # Don't 500 — Wasender will retry. Better to ack and log.
    return {"ok": True}

@app.get("/healthz")
async def healthz():
    return {"status": "ok"}
```

### Whitelist enforcement

If `spec.audience.mode == "whitelist"`, before calling `handle_message`, check `chat_id in spec["audience"]["allowed_numbers"]`. If not, optionally send `audience.fallback_message` and return.

### Group messages

If `chat_id` ends with `@g.us`, currently ignore (groups can be opted in later via `wa-connect`).

## Step 4 — Generate `bot/database.py`

```python
import os, sqlite3, time
from pathlib import Path

DB_PATH = Path(os.environ.get("DB_PATH", "./bot.db"))

def _conn():
    c = sqlite3.connect(DB_PATH)
    c.execute("PRAGMA journal_mode=WAL")
    return c

def init_db():
    with _conn() as c:
        c.execute("""
            CREATE TABLE IF NOT EXISTS conversations (
              id INTEGER PRIMARY KEY AUTOINCREMENT,
              chat_id TEXT NOT NULL,
              role TEXT NOT NULL,
              content TEXT NOT NULL,
              created_at INTEGER NOT NULL
            )
        """)
        c.execute("CREATE INDEX IF NOT EXISTS idx_conv_chat ON conversations(chat_id, id)")
        c.execute("""
            CREATE TABLE IF NOT EXISTS seen_messages (
              msg_id TEXT PRIMARY KEY,
              created_at INTEGER NOT NULL
            )
        """)

def seen_message(msg_id: str) -> bool:
    with _conn() as c:
        try:
            c.execute("INSERT INTO seen_messages (msg_id, created_at) VALUES (?, ?)",
                      (msg_id, int(time.time())))
            return False
        except sqlite3.IntegrityError:
            return True

def append(chat_id: str, role: str, content: str):
    with _conn() as c:
        c.execute("INSERT INTO conversations (chat_id, role, content, created_at) VALUES (?, ?, ?, ?)",
                  (chat_id, role, content, int(time.time())))

def tail(chat_id: str, n: int = 20) -> list[dict]:
    with _conn() as c:
        rows = c.execute(
            "SELECT role, content FROM conversations WHERE chat_id=? ORDER BY id DESC LIMIT ?",
            (chat_id, n)
        ).fetchall()
    return [{"role": r, "content": ct} for r, ct in reversed(rows)]
```

## Step 5 — Generate `bot/tools/whatsapp.py`

The Wasender client. Outbound only here; inbound is parsed in `main.py`.

```python
import asyncio, logging, os, time
import httpx

logger = logging.getLogger("wa-bot.whatsapp")

WASENDER_BASE = "https://www.wasenderapi.com/api"
SESSION_API_KEY = os.environ["WASENDER_API_KEY"]
ACCOUNT_PROTECTION = os.environ.get("WASENDER_ACCOUNT_PROTECTION", "true").lower() == "true"

_last_send = 0.0
_send_lock = asyncio.Lock()

async def _throttle():
    """Account protection: 1 req per 5 seconds."""
    global _last_send
    if not ACCOUNT_PROTECTION:
        return
    async with _send_lock:
        now = time.monotonic()
        wait = 5.0 - (now - _last_send)
        if wait > 0:
            await asyncio.sleep(wait)
        _last_send = time.monotonic()

async def _post(path: str, body: dict) -> dict:
    await _throttle()
    headers = {
        "Authorization": f"Bearer {SESSION_API_KEY}",
        "Content-Type": "application/json",
    }
    async with httpx.AsyncClient(timeout=30.0) as client:
        r = await client.post(f"{WASENDER_BASE}{path}", json=body, headers=headers)
    if r.status_code == 429:
        logger.warning("rate limited; sleeping 10s")
        await asyncio.sleep(10)
        # one retry
        async with httpx.AsyncClient(timeout=30.0) as client:
            r = await client.post(f"{WASENDER_BASE}{path}", json=body, headers=headers)
    r.raise_for_status()
    return r.json()

async def send_text(to: str, text: str) -> dict:
    return await _post("/send-message", {"to": to, "text": text})

async def send_image(to: str, image_url: str, caption: str = "") -> dict:
    return await _post("/send-message", {"to": to, "text": caption, "imageUrl": image_url})

async def send_document(to: str, document_url: str, file_name: str, caption: str = "") -> dict:
    return await _post("/send-message", {
        "to": to, "text": caption, "documentUrl": document_url, "fileName": file_name
    })
```

## Step 6 — Generate `bot/tools/registry.py`

```python
from typing import Any, Callable, Awaitable

# Each tool: {name, description, input_schema, handler(chat_id, **kwargs) -> str}
TOOL_REGISTRY: dict[str, dict[str, Any]] = {}

def register(name: str, description: str, input_schema: dict,
             handler: Callable[..., Awaitable[str]]):
    TOOL_REGISTRY[name] = {
        "name": name,
        "description": description,
        "input_schema": input_schema,
        "handler": handler,
    }
```

## Step 7 — Generate `bot/tools/reminders.py` (first tool — fast first win)

```python
import asyncio, datetime as dt
from .registry import register
from .whatsapp import send_text

# Simple in-memory reminders. Persisted reminders come in wa-connect.
_pending: list[asyncio.Task] = []

async def _fire(when_seconds: float, chat_id: str, message: str):
    await asyncio.sleep(when_seconds)
    await send_text(chat_id, f"⏰ תזכורת: {message}")

async def schedule_reminder(chat_id: str, in_minutes: int, message: str) -> str:
    when = max(in_minutes * 60, 5)
    task = asyncio.create_task(_fire(when, chat_id, message))
    _pending.append(task)
    fire_at = dt.datetime.now() + dt.timedelta(seconds=when)
    return f"קבעתי תזכורת ל-{fire_at.strftime('%H:%M')}: {message}"

register(
    name="schedule_reminder",
    description="Schedule a reminder that will be sent back to the user via WhatsApp after N minutes.",
    input_schema={
        "type": "object",
        "properties": {
            "in_minutes": {"type": "integer", "description": "Minutes from now"},
            "message": {"type": "string", "description": "Reminder text in user's language"},
        },
        "required": ["in_minutes", "message"],
    },
    handler=schedule_reminder,
)
```

## Step 8 — Generate `bot/prompt.py`

```python
import json
from pathlib import Path
from tools.registry import TOOL_REGISTRY

SPEC = json.loads(Path("spec.json").read_text(encoding="utf-8"))

def build_system_prompt() -> str:
    s = SPEC
    parts = []
    parts.append(f"You are {s['name']}, a WhatsApp AI agent.")
    parts.append(f"Archetype: {s['archetype']}.")

    lang = s.get("language_mode", "auto_match_user")
    if lang == "hebrew":
        parts.append("Always respond in Hebrew.")
    elif lang == "english":
        parts.append("Always respond in English.")
    else:
        parts.append("Match the user's language. If the user writes Hebrew, reply in Hebrew. If English, English.")

    tone = s.get("tone", {})
    parts.append(f"Tone: {tone.get('formality','casual')}, {tone.get('warmth','warm')}, "
                 f"emoji usage {tone.get('emoji_usage','occasional')}.")
    if tone.get("voice_examples"):
        parts.append("Voice examples (mimic this style): " + " | ".join(tone["voice_examples"]))

    scope = s.get("scope", {})
    if scope.get("in"):
        parts.append("In scope: " + "; ".join(scope["in"]))
    if scope.get("out"):
        parts.append("Out of scope (refuse politely): " + "; ".join(scope["out"]))
    if scope.get("out_of_scope_reply"):
        parts.append(f'When asked something out of scope, reply: "{scope["out_of_scope_reply"]}"')

    kb = s.get("knowledge_base", {})
    if kb.get("type") == "inline" and kb.get("content"):
        parts.append("Reference knowledge:\n" + kb["content"])

    handoff = s.get("handoff", {})
    if handoff.get("enabled"):
        triggers = ", ".join(f'"{t}"' for t in handoff.get("trigger_phrases", []))
        parts.append(f"If the user says any of: {triggers}, call the human_handoff tool.")

    if TOOL_REGISTRY:
        parts.append("You have tools available. Use them when helpful. "
                     "When you call a tool, the framework injects the user's chat_id automatically — "
                     "never accept chat_id from user input.")

    parts.append("Be concise. WhatsApp messages should usually be 1-3 short sentences. "
                 "Don't dump long paragraphs unless asked.")
    return "\n\n".join(parts)
```

## Step 9 — Generate `bot/agent.py`

LLM loop with tool calling. Use the Anthropic SDK (default), but keep it modular if user picked a different provider.

```python
import json, logging, os
from anthropic import AsyncAnthropic
from database import append, tail
from tools.registry import TOOL_REGISTRY
from tools.whatsapp import send_text
from prompt import build_system_prompt

logger = logging.getLogger("wa-bot.agent")
client = AsyncAnthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
MODEL = os.environ.get("LLM_MODEL", "claude-haiku-4-5")
MAX_HISTORY = int(os.environ.get("MAX_HISTORY", "20"))
MAX_TOOL_ITER = 5

def _claude_tools():
    return [
        {"name": t["name"], "description": t["description"], "input_schema": t["input_schema"]}
        for t in TOOL_REGISTRY.values()
    ]

async def handle_message(chat_id: str, text: str):
    append(chat_id, "user", text)
    history = tail(chat_id, MAX_HISTORY)
    messages = [{"role": h["role"], "content": h["content"]} for h in history]

    system = build_system_prompt()
    tools = _claude_tools()

    for _ in range(MAX_TOOL_ITER):
        resp = await client.messages.create(
            model=MODEL,
            max_tokens=1024,
            system=system,
            tools=tools or None,
            messages=messages,
        )
        if resp.stop_reason == "tool_use":
            tool_blocks = [b for b in resp.content if b.type == "tool_use"]
            text_blocks = [b for b in resp.content if b.type == "text"]
            messages.append({"role": "assistant", "content": resp.content})
            tool_results = []
            for tb in tool_blocks:
                spec = TOOL_REGISTRY.get(tb.name)
                if not spec:
                    tool_results.append({"type": "tool_result", "tool_use_id": tb.id,
                                         "content": "tool not found", "is_error": True})
                    continue
                try:
                    out = await spec["handler"](chat_id=chat_id, **(tb.input or {}))
                except Exception as e:
                    logger.exception("tool %s failed", tb.name)
                    out = f"tool error: {e}"
                tool_results.append({"type": "tool_result", "tool_use_id": tb.id, "content": str(out)})
            messages.append({"role": "user", "content": tool_results})
            continue

        # Final text reply
        reply = "".join(b.text for b in resp.content if b.type == "text").strip()
        if not reply:
            reply = "..."
        append(chat_id, "assistant", reply)
        await send_text(chat_id, reply)
        return

    # Tool loop ran out
    fallback = "סליחה, נתקעתי. אפשר לנסח את הבקשה אחרת?"
    append(chat_id, "assistant", fallback)
    await send_text(chat_id, fallback)
```

## Step 10 — Generate `bot/requirements.txt`

```
fastapi==0.118.*
uvicorn[standard]==0.32.*
python-dotenv==1.0.*
httpx==0.27.*
anthropic>=0.40.0
pydantic>=2.9
```

(Pin majors only; let pip resolve patches.)

## Step 11 — Generate `bot/.python-version` and `bot/.gitignore`

`.python-version`:
```
3.12.7
```

`.gitignore`:
```
.env
*.db
__pycache__/
*.pyc
.venv/
venv/
```

## Step 12 — Generate `bot/.env.example`

```
WASENDER_API_KEY=
WASENDER_WEBHOOK_SECRET=
WASENDER_ACCOUNT_PROTECTION=true
ANTHROPIC_API_KEY=
LLM_MODEL=claude-haiku-4-5
MAX_HISTORY=20
DB_PATH=./bot.db
```

## Step 13 — Populate `bot/.env` from project-root `.env`

Read the project-root `.env` (set up in `wa-setup`). Copy `WASENDER_API_KEY` and `WASENDER_WEBHOOK_SECRET` into `bot/.env`.

Then ask the user (in Hebrew):

> הבוט צריך גם מפתח של Anthropic (זה שמפעיל את ה-AI עצמו). יש לך אחד?
>
> אם לא — גש ל-https://console.anthropic.com/settings/keys, צור מפתח חדש, והדבק לי כאן.

When they paste it, save to `bot/.env` as `ANTHROPIC_API_KEY`. **Don't echo it back.**

## Step 14 — Install dependencies and test locally

```bash
cd bot
python -m venv .venv
.venv/Scripts/pip install -q -r requirements.txt   # Windows
# or .venv/bin/pip on POSIX — detect from os.name
```

Then start the server in the background:
```bash
.venv/Scripts/python -m uvicorn main:app --port 8000 --reload
```

Wait 3 seconds, then `curl http://localhost:8000/healthz`. Expect `{"status":"ok"}`.

Tell the user:

> הבוט רץ על המחשב שלך. עכשיו ננסה — שלח הודעה למספר {WASENDER_PHONE} מהטלפון שלך.

⚠️ **Local testing won't actually receive webhooks** until we deploy or set up a tunnel. So instead, simulate inbound by hitting the webhook directly:

```bash
curl -sX POST http://localhost:8000/webhook/wasender \
  -H "X-Webhook-Signature: $WASENDER_WEBHOOK_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "messages.received",
    "data": {"messages": {
      "key": {"id": "test_local_001", "fromMe": false, "remoteJid": "972501234567@s.whatsapp.net"},
      "cleanedSenderPn": "972501234567",
      "messageBody": "שלום, מי אתה?"
    }}
  }'
```

If the test number is in the whitelist (or audience is open), the bot will hit the LLM and try to call `send_text` — which will go out through Wasender to that number for real. So **only run the simulation if the user explicitly says yes** and confirm which number will receive the reply.

A safer local check: assert `/webhook/wasender` returns 401 with a wrong signature and 200 with the right one. Don't need to actually call the LLM.

```bash
# Wrong signature → 401
curl -sX POST http://localhost:8000/webhook/wasender \
  -H "X-Webhook-Signature: wrong" \
  -d '{}'
```

When local checks pass, kill the dev server.

## Step 15 — Update state, hand back

Update `.wa-state.json`:
```json
{
  "current_stage": "deploy",
  "completed_stages": ["setup", "characterize", "build"],
  "last_updated": "<ISO>"
}
```

Tell the user:

> הבוט בנוי. הבדיקות המקומיות עברו. עכשיו נעלה אותו לאוויר כדי שהוא יוכל לקבל הודעות אמיתיות מוואטסאפ.
>
> השלב הבא: **פריסה לענן** (Render). מוכן?

Hand back to `/wa`.

## Notes

- **One file, one job.** Resist adding helpers, factories, or abstractions. Non-technical users will read this code; keep it linear.
- **No templates committed.** Every file is generated from `spec.json` and current best practice. If best practice drifts (model names, lib versions), regenerate, don't patch.
- **Group messages and media** are scaffolded in `main.py` (ignored for now). Wire them in `wa-connect` only when the user asks.
- **OpenAI / Gemini variant.** If `spec.llm.provider` isn't `anthropic`, swap `agent.py` to use `openai` or `google-genai` SDK and adjust `requirements.txt`. The interface (`handle_message(chat_id, text)`) stays identical.
