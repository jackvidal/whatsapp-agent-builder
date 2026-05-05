---
name: wa-maintain
description: Forever-stage. Tweak the bot's personality, refresh expired tokens, diagnose "the bot is acting weird", recover from a disconnected WhatsApp, update tools, view recent conversations. The user lives here once the bot is deployed.
---

# wa-maintain — Keep the bot healthy

Goal: be the user's on-call engineer. They tell you what's wrong (or what they want to change) and you handle it. **Always diagnose before changing code.**

**Speak Hebrew. Be calm. Most issues have a 30-second fix.**

## Greet based on what they said

When `/wa` routes here, ask:

> במה אפשר לעזור?
>
> 🔧 הבוט עונה משהו מוזר / לא עונה / ענה דבר שאסור לו
> ✏️ לשנות את האישיות, הטון, או התשובות
> 🔌 הוואטסאפ התנתק / קוד QR פג
> 🛠️ לעדכן או להחליף יכולת קיימת
> 📜 לראות שיחות אחרונות
> 🚫 לחסום מספר / לפתוח לכולם
> אחר — תכתוב במילים שלך

## Diagnostic ladder — when something is broken

When the user says "הבוט לא עונה" or similar, run this 7-step ladder **in order**. Stop at the first failure and fix it.

### Step 1 — Wasender session status
```bash
curl -sX GET "https://www.wasenderapi.com/api/status" \
  -H "Authorization: Bearer $WASENDER_API_KEY"
```
- `connected` ✅ → step 2.
- `need_scan` / `disconnected` → re-run `wa-setup` step 4 (fetch QR, render image, user scans).
- `expired` / `logged_out` → user logged the bot out from their phone's "Linked Devices". Tell them, re-scan.

### Step 2 — Render service is live and recently deployed
```bash
curl -s "https://api.render.com/v1/services/$RENDER_SERVICE_ID" \
  -H "Authorization: Bearer $RENDER_API_KEY"
```
- `status: live` ✅ → step 3.
- `status: build_failed` / `update_failed` → fetch the latest deploy events, surface error in Hebrew, fix.
- `status: suspended` → free tier hit usage limits. Tell user.

### Step 3 — Health endpoint responds
```bash
curl -s "$RENDER_URL/healthz"
```
- `{"status":"ok"}` ✅ → step 4.
- Empty / 502 → service is asleep (Render free tier). Wait 30s, retry. If still failing: check Render logs.

### Step 4 — Webhook URL registered correctly with Wasender

**Branch on `setup_path`** from `.wa-state.json`:

**Path A (PAT available):**
```bash
curl -s "https://www.wasenderapi.com/api/whatsapp-sessions/$WASENDER_SESSION_ID" \
  -H "Authorization: Bearer $WASENDER_PAT" | jq '{webhook_url, webhook_enabled, webhook_events}'
```
Expect `webhook_url == $RENDER_URL + "/webhook/wasender"`, `webhook_enabled: true`, `messages.received` in events.

If wrong: re-register (`PUT /api/whatsapp-sessions/{id}` with the right values — see `wa-deploy` step 7).

**Path B (no PAT):**
Ask the user to confirm visually in the dashboard:
> תפתח את הסשן ב-https://wasenderapi.com/dashboard, תלחץ **Manage Webhook**. ותגיד לי:
> 1. מה כתוב ב-Webhook URL?
> 2. מסומן `messages.received`?

Compare against `RENDER_URL + "/webhook/wasender"`. If mismatched, walk them through fixing it (same instructions as `wa-deploy` step 7 Path B).

### Step 5 — Webhook signature matches
The most common silent failure: env var `WASENDER_WEBHOOK_SECRET` on Render doesn't match what Wasender is sending.

Trigger a controlled test: send "test" from any whitelisted number, then immediately:
```bash
curl -s "https://api.render.com/v1/services/$RENDER_SERVICE_ID/logs?limit=20" \
  -H "Authorization: Bearer $RENDER_API_KEY"
```

Look for "bad signature" 401s. If found: fetch the current secret from Wasender:
```bash
curl -s "https://www.wasenderapi.com/api/whatsapp-sessions/$WASENDER_SESSION_ID" \
  -H "Authorization: Bearer $WASENDER_PAT" | jq -r '.data.webhook_secret'
```

Update Render env `WASENDER_WEBHOOK_SECRET` and redeploy.

### Step 6 — Inbound messages are reaching the webhook
Check Render logs for recent POSTs to `/webhook/wasender` with 200 responses. If nothing arriving despite step 4+5 being correct: the user might be messaging the wrong number, or the session got muted by Wasender. Check session-logs:
```bash
curl -s "https://www.wasenderapi.com/api/whatsapp-sessions/$WASENDER_SESSION_ID/session-logs?limit=20" \
  -H "Authorization: Bearer $WASENDER_PAT"
```

### Step 6.5 — Sender arrived as a LID (modern WhatsApp privacy mode)

If logs show `inbound msg_id=... remoteJid=...@lid cleanedSenderPn=None`, you've hit WhatsApp's LID privacy model. Symptoms:
- Whitelist mode rejects the user even when their phone *should* be allowed
- The fallback "you're not whitelisted" send 422s with `"The provided JID does not exist on WhatsApp."` because the bot is trying to send to `+<lid>` which isn't a real phone

**Why it happens.** WhatsApp 2024-2025 anonymizes sender phone numbers behind LIDs (Linked-device IDs) when the sender hasn't saved the recipient's number. Wasender (Baileys-based) only receives what WhatsApp gives it.

**Fix in code (should already be in `wa-build`'s generated `main.py`):**
1. Use `remote_jid` as `chat_id` when `cleanedSenderPn` is `None` — preserves the `@lid` suffix.
2. `is_allowed()` accepts either the JID *or* the `+phone` form for matching.
3. Wasender's `/send-message` accepts both formats in `to` — sending to `<lid>@lid` works.

**Workarounds if the user wants whitelist-only:**
- Paste the LID into `spec.audience.allowed_numbers` (e.g. `"124833811697866@lid"`) — works but is opaque
- Have the user save the bot's number in their phone contacts; sometimes WhatsApp then promotes future messages back to phone-mode (not guaranteed)
- Switch to `audience.mode: "open"` and rely on the LLM's scope rules to decline non-target users

### Step 7 — LLM call is succeeding
Check Render logs for `anthropic` / `openai` errors:
- 401 → API key invalid / billing issue.
- 429 → rate-limited.
- 5xx → provider down; nothing we can do but retry.
- Tool error → look at the specific tool that failed.

## Tweaking personality / scope

User says "שנה את הטון להיות יותר רשמי" or "אל תאפשר לו לדבר על מחירים":

1. Read `bot/spec.json`.
2. Modify the relevant field (`tone.formality`, `scope.out`, etc.).
3. Save back.
4. **No code change needed** — `prompt.py` reads `spec.json` at process start.
5. To pick up the change live, restart the Render service (or bump a deploy):
```bash
curl -sX POST "https://api.render.com/v1/services/$RENDER_SERVICE_ID/deploys" \
  -H "Authorization: Bearer $RENDER_API_KEY" \
  -H "Content-Type: application/json" -d '{"clearCache": "do_not_clear"}'
```

Confirm with user:
> שיניתי. בעוד דקה הבוט יעבוד עם הטון החדש. שלח לו הודעה לבדוק.

## Refreshing OAuth tokens (Calendar / Gmail)

Symptom: tool returns "credentials expired" or 401 from Google.

1. Visit `{RENDER_URL}/auth/google` from the user's browser.
2. Grant permissions again.
3. New refresh token saved automatically.

If the user revoked access in their Google account, the refresh token won't work — they need to re-auth fresh.

## Viewing recent conversations

The user might want to see what the bot said to a specific contact (audit). If persistence is SQLite on disk:

```bash
# Pull the bot.db from Render via shell session, or:
# Add a small password-protected endpoint /admin/conversations?chat_id=+972...
```

For non-technical users, easier: write a tiny one-off Python script locally that queries the DB and prints the last N turns. Or better — add a `/admin/conversations` route to `main.py` that requires a basic-auth token and returns HTML.

## Reconnecting WhatsApp after disconnect

If WhatsApp pulls the linked device (the user logged out from their phone, or hit the 14-day inactivity limit):

1. Run `wa-setup` step 4 (connect + fetch QR + render image).
2. User scans.
3. Status returns to `connected`.

The `WASENDER_API_KEY` and `WASENDER_WEBHOOK_SECRET` **don't change** on reconnection — same session, same credentials. No Render redeploy needed.

## Updating a tool

User says "Gmail מתחיל לרוץ אבל לא שולח":

1. Check Render logs around the failure timestamp.
2. Identify the exception.
3. Fix the tool file in `bot/tools/<tool>.py`.
4. Push to GitHub. Render auto-deploys.
5. Verify with a test message.

Don't refactor adjacent code while you're in there. Surgical fixes only.

## Removing a tool

```bash
cd bot
git rm tools/<tool>.py
git commit -m "remove tool: <tool>"
git push
```

Update `.wa-state.connected_tools` (remove the entry). The auto-import in `tools/__init__.py` won't see the missing file; the LLM won't be offered the tool.

## Bot is too slow / responses are cut off

- Increase `MAX_HISTORY` env var on Render (default 20 → try 40). Trade-off: more tokens, more cost per message.
- Increase `max_tokens` in `agent.py` (default 1024 → 2048). Same trade-off.
- If `account_protection: true` is making sends slow: in production with a known good number, you can disable it (`WASENDER_ACCOUNT_PROTECTION=false` env). Warn user about ban risk.

## Token / API key rotated

User regenerated their Wasender PAT, Anthropic key, etc.:
1. Update project-root `.env` (locally) and Render env vars (remote).
2. Redeploy if it's an env Render needs.
3. For PAT: the PAT only matters for the plugin's own session-management calls (`wa-setup`, `wa-maintain`, `wa-deploy`). The runtime bot uses `WASENDER_API_KEY` (per-session), which doesn't rotate unless the user explicitly regenerates it from the Wasender dashboard.

## What never to do

- **Don't blindly redeploy** when something's broken. Diagnose first — redeploy is rarely the fix and slows iteration.
- **Don't paste secrets** into the chat as you debug. Refer to env var names.
- **Don't edit code on Render's web shell.** Always commit + push so changes survive redeploys.
- **Don't rebuild from scratch** if one tool is broken. Fix the tool.
