# WhatsApp Agent Builder by Jack Vidal

> **Build, deploy & maintain your own WhatsApp AI agent — no coding required.**
> Hebrew-first. Wasender-powered. Runs on Claude Code.

A Claude Code plugin that walks non-technical users through the entire lifecycle of a WhatsApp AI agent, in plain Hebrew, one decision at a time. You make the choices; Claude Code does the work.

---

## What it builds

A live WhatsApp number that:
- Replies to messages with Claude (or OpenAI / Gemini) as its brain
- Has a personality, scope, and audience you define
- Can integrate with Google Calendar, Gmail, persistent reminders, human handoff, group chat, and voice transcription
- Runs 24/7 on Render.com (free tier works for personal use)
- Stores conversation history in SQLite or Postgres

Stack of the generated bot: **Python 3.12 + FastAPI + httpx + Anthropic SDK**, deployed to **Render.com**, connected to WhatsApp via **[Wasender](https://wasenderapi.com)**.

## Install

```bash
# In Claude Code:
/plugin marketplace add jackvidal/whatsapp-agent-builder
/plugin install whatsapp-agent-builder@jack-vidal-plugins
```

Then start the flow:

```
/wa
```

That's it. Claude Code reads `.wa-state.json` in the current directory, figures out what stage you're at, and routes you to the right skill.

## The seven stages

1. **`wa-setup`** — Sign up at Wasender, generate a Personal Access Token, create a WhatsApp session, scan a QR with your phone, verify the connection.
2. **`wa-characterize`** — A short Hebrew Q&A that produces `spec.json`: name, tone, audience whitelist, in/out scope, knowledge base, tools wishlist.
3. **`wa-build`** — Generates the entire FastAPI bot from `spec.json` (no templates — fresh code traces directly to your spec).
4. **`wa-deploy`** — Pushes to GitHub, deploys to Render, registers the webhook URL with Wasender, runs a real round-trip test.
5. **`wa-connect`** — Adds tools one at a time: Calendar, Gmail, persistent reminders, human handoff, groups, voice notes.
6. **`wa-persistence`** — Choose where the bot's memory lives: ephemeral SQLite, Render Disk, Supabase, or Render Postgres.
7. **`wa-maintain`** — Forever-stage. Tweak personality, refresh tokens, diagnose "the bot is acting weird," reconnect after disconnect.

The plugin tracks state in `.wa-state.json` so you can stop and resume any time.

## Why Wasender (and the trade-offs)

Wasender is a Baileys-based WhatsApp gateway — **not the official WhatsApp Business Cloud API**. That means:

✅ **Cheap** — $6/month for 1 number, no per-message fees, 3-day free trial.
✅ **Simple** — no Meta Business verification, no 24-hour reply window, full inbox visibility.
✅ **Fast to launch** — connect via QR scan in under a minute.

❌ **Unofficial** — there's a theoretical risk of WhatsApp banning the connected number. Always enable `account_protection: true` (the plugin defaults this on) and use a dedicated/secondary number for production.
❌ **Rate-limited** — with account protection: 1 message per 5 seconds. Fine for personal/small-business, not for mass marketing (which you shouldn't do anyway).

For most personal-assistant and small-business use cases, this is the right trade-off.

## Design principles

- **"I do, you decide."** Claude Code runs every command, edits every file, makes every API call. The user makes decisions.
- **Hebrew throughout the user-facing dialog.** Internal architectural notes stay in English.
- **No templates.** Every file is generated from the user's `spec.json`, so reading the code teaches you what each choice did.
- **Diagnose before changing.** `wa-maintain` has a 7-step ladder for debugging. Redeploy is rarely the fix.
- **Fast first win.** `wa-build` wires only one tool (reminders) initially, so the bot replies on Wasender end-to-end before any OAuth complexity.

## Repo structure

```
whatsapp-agent-builder/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/
│   └── wa.md                     # /wa orchestrator
├── skills/
│   ├── wa-setup/SKILL.md
│   ├── wa-characterize/SKILL.md
│   ├── wa-build/SKILL.md
│   ├── wa-deploy/SKILL.md
│   ├── wa-connect/SKILL.md
│   ├── wa-persistence/SKILL.md
│   └── wa-maintain/SKILL.md
├── .env.example
├── .gitignore
├── LICENSE
└── README.md
```

The plugin generates a separate `bot/` folder in the user's project directory — that's the actual runtime code that gets deployed.

## Costs

Realistic monthly cost for a personal WhatsApp agent:
- **Wasender:** $6 (Basic, 1 number)
- **Render:** $0 (free tier, sleeps after 15min idle) or $7 (always-on)
- **Anthropic API:** $1–5 (Haiku 4.5 for typical personal-assistant usage)
- **Supabase / Postgres:** $0 (free tier) or $25 (Supabase Pro)

So roughly **$7–18/month** for a hobby bot, **$30–50/month** for an always-on production bot.

## Credits

- Architectural inspiration: [Asher-pro/wa-whatsapp-agent](https://github.com/Asher-pro/wa-whatsapp-agent) (the GreenAPI version)
- WhatsApp gateway: [Wasender](https://wasenderapi.com)
- Built for [Claude Code](https://claude.com/claude-code)

## License

MIT — see [LICENSE](./LICENSE).

---

**Author:** Jack Vidal · [jack@jackvidal.com](mailto:jack@jackvidal.com)
