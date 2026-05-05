---
name: wa-persistence
description: Stage 6 (optional). Choose where conversation history and OAuth tokens live. Default is SQLite on Render's ephemeral disk; production needs Render Disk, Supabase, or Render Postgres.
---

# wa-persistence — Where the bot's memory lives

Goal: pick the right storage backend for the user's situation. Default is fine for most personal bots; production / shared bots need an upgrade.

**Speak Hebrew. Explain trade-offs in plain language. Don't push them to the most complex option.**

## The four options

### 1. SQLite ephemeral (default — what `wa-build` ships)
- File at `./bot.db` on Render's filesystem.
- ✅ Zero setup, zero cost.
- ❌ **Wiped on every redeploy** on Render free tier (free tier has no persistent disk).
- ❌ Wiped on container restart.
- 🟡 Works for testing, demos, "I'll redo this later."

### 2. SQLite on Render Persistent Disk ($1/mo)
- Same SQLite file, but mounted on a 1GB persistent disk.
- ✅ Survives redeploys and restarts.
- ✅ Same simple code as default — no migration.
- ❌ Disk only attaches to a single instance. If Render scales horizontally (paid tiers), only one container can write.
- 🟡 Right answer for personal bots and small businesses.

### 3. Supabase Postgres (free tier, then $25/mo)
- Managed Postgres, accessed via `DATABASE_URL`.
- ✅ Free tier (500 MB, paused after 7 days idle — wakes on demand).
- ✅ Survives everything. Visible in Supabase UI for debugging.
- ❌ Requires external account, slight extra latency (~30ms).
- 🟡 Right answer if user already uses Supabase, or wants to share data with another app.

### 4. Render Postgres ($7/mo)
- Render-native Postgres, internal network (low latency).
- ✅ Same region as the bot, fast.
- ✅ Daily backups.
- ❌ Costs from day one.
- 🟡 Right answer for a paid bot that needs reliability.

## The conversation

> רוצה להחליט איפה הבוט שומר את ההיסטוריה של השיחות (וההרשאות לכלים החיצוניים)?
>
> כרגע הוא שומר בקובץ זמני שנמחק כשהבוט מתעדכן. בשבילך זה יכול להיות בסדר, או לא — תלוי במה שאתה עושה איתו.
>
> 4 אפשרויות:
>
> **א. הזמני שיש כרגע** — חינם, נמחק בכל עדכון. טוב לבדיקות.
> **ב. דיסק מתמיד ב-Render** — $1 בחודש. נשמר תמיד. הכי קל.
> **ג. Supabase** — חינם בהתחלה, $25 כשגדל. שירות מסודר עם ממשק לבדיקת הנתונים.
> **ד. Postgres ב-Render** — $7 בחודש. הכי מהיר ויציב, עם גיבויים.
>
> מה מתאים לך?

Default recommendation per archetype (offer it but let user override):
- `personal_assistant` → ב (Render Disk)
- `customer_support` → ד (Render Postgres)
- `knowledge_bot` → ב (Render Disk)
- `scheduler` → ד (Render Postgres) if multi-tenant, else ב

## Migration steps

### Option ב — Render Persistent Disk

1. Add a disk to the existing Render service via API:
```bash
curl -sX POST "https://api.render.com/v1/services/$RENDER_SERVICE_ID/disks" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "bot-data", "mountPath": "/var/data", "sizeGB": 1}'
```

2. Update Render env: `DB_PATH=/var/data/bot.db`.
3. Trigger a redeploy.
4. Verify: `curl $RENDER_URL/healthz` after deploy completes.

The history will be empty after the first migrated deploy (the old disk is gone). Tell the user:
> ההיסטוריה הקיימת תיעלם בעדכון הזה (לא היה איפה לשמור אותה). מהיום והלאה — נשמר.

### Option ג — Supabase

1. Walk user through:
   - https://supabase.com/dashboard → New Project (Free tier)
   - Wait ~2min for provision
   - Settings → Database → Connection string → URI (with password)

2. Save `DATABASE_URL` as Render env var.

3. Update `bot/database.py`: replace `sqlite3` with `psycopg[binary]`. Provide drop-in module:

```python
import os, time
import psycopg
from psycopg.rows import dict_row

_dsn = os.environ["DATABASE_URL"]

def init_db():
    with psycopg.connect(_dsn) as c, c.cursor() as cur:
        cur.execute("""
          CREATE TABLE IF NOT EXISTS conversations (
            id BIGSERIAL PRIMARY KEY,
            chat_id TEXT NOT NULL,
            role TEXT NOT NULL,
            content TEXT NOT NULL,
            created_at BIGINT NOT NULL
          );
          CREATE INDEX IF NOT EXISTS idx_conv_chat ON conversations(chat_id, id);
          CREATE TABLE IF NOT EXISTS seen_messages (
            msg_id TEXT PRIMARY KEY,
            created_at BIGINT NOT NULL
          );
        """)

def seen_message(msg_id: str) -> bool:
    with psycopg.connect(_dsn) as c, c.cursor() as cur:
        try:
            cur.execute("INSERT INTO seen_messages (msg_id, created_at) VALUES (%s, %s)",
                        (msg_id, int(time.time())))
            return False
        except psycopg.errors.UniqueViolation:
            return True

def append(chat_id: str, role: str, content: str):
    with psycopg.connect(_dsn) as c, c.cursor() as cur:
        cur.execute("INSERT INTO conversations (chat_id, role, content, created_at) "
                    "VALUES (%s, %s, %s, %s)", (chat_id, role, content, int(time.time())))

def tail(chat_id: str, n: int = 20) -> list[dict]:
    with psycopg.connect(_dsn) as c, c.cursor(row_factory=dict_row) as cur:
        cur.execute("SELECT role, content FROM conversations WHERE chat_id=%s "
                    "ORDER BY id DESC LIMIT %s", (chat_id, n))
        rows = cur.fetchall()
    return list(reversed(rows))
```

4. Add `psycopg[binary]==3.2.*` to `requirements.txt`. Remove sqlite-only code paths.
5. Push, wait for redeploy, verify.

### Option ד — Render Postgres

1. Create the Postgres instance via Render API:
```bash
curl -sX POST "https://api.render.com/v1/postgres" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "wa-bot-db", "ownerId": "'$RENDER_OWNER_ID'", "plan": "starter", "region": "frankfurt", "version": "16"}'
```

2. Wait for `status: available`, fetch the internal connection string from `GET /v1/postgres/{id}/connection-info`.

3. Set `DATABASE_URL` env on the bot service. Same `database.py` swap as Supabase. Push, redeploy, verify.

## After migration

Update `.wa-state.json`:
```json
{ "persistence": "render_disk" | "supabase" | "render_postgres" | "ephemeral" }
```

Tell the user:

> ההיסטוריה כעת נשמרת ב-{choice}. הבוט יזכור שיחות גם אחרי עדכונים.

## Edge cases

- **User picked option ב after already running for weeks.** Old SQLite data lives on the ephemeral disk only until next redeploy. If they want to preserve it: scp the file out before adding the disk, then put it back at `/var/data/bot.db` on first boot. Walk them through, but for non-technical users it's usually fine to start fresh.
- **Supabase paused.** Free Supabase pauses after 7 days idle. First message after pause takes ~30s to wake. Document this.
- **Migration to a different option later.** Always possible. Conversation table schema is identical across all options. OAuth tokens table will need migrating too — write a one-time script if user switches.
