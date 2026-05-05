---
name: wa-characterize
description: Stage 2. Hebrew Q&A that produces spec.json — the bot's name, tone, audience, in/out scope, knowledge base, and tools wishlist. This spec drives every later stage.
---

# wa-characterize — Define the bot's personality and scope

Goal: produce a structured `spec.json` file. Every later stage (build, connect, deploy, maintain) reads from `spec.json` to know what the bot is and what it should do. Get it right here and the rest is automatic.

**Speak Hebrew. Ask questions one at a time, not all at once.** Non-technical users get overwhelmed by long forms. Conversational beats checklist.

## The output: spec.json

By the end, write `./spec.json`:

```json
{
  "version": 1,
  "name": "...",
  "archetype": "personal_assistant | customer_support | knowledge_bot | scheduler",
  "language_mode": "hebrew | english | auto_match_user",
  "audience": {
    "mode": "whitelist | open",
    "allowed_numbers": ["+972..."],
    "fallback_message": "..."
  },
  "tone": {
    "formality": "casual | semi_formal | formal",
    "warmth": "warm | neutral | brisk",
    "emoji_usage": "frequent | occasional | none",
    "voice_examples": ["...", "..."]
  },
  "scope": {
    "in": ["..."],
    "out": ["..."],
    "out_of_scope_reply": "..."
  },
  "knowledge_base": {
    "type": "none | inline | files",
    "content": "..." 
  },
  "tools_wishlist": ["calendar", "gmail", "reminders", "human_handoff"],
  "handoff": {
    "enabled": false,
    "owner_number": null,
    "trigger_phrases": []
  },
  "llm": {
    "provider": "anthropic",
    "model": "claude-haiku-4-5",
    "max_history": 20
  }
}
```

## The conversation flow

Open with:

> נצטרך עכשיו להחליט מה הבוט שלך עושה ואיך הוא נשמע. אני שואל, אתה עונה. אין תשובות נכונות וטעויות — רק העדפות שלך.

### Q1 — Archetype (the easy first decision)

> מה הבוט הזה צריך להיות, בגדול?
>
> 1️⃣ **עוזר אישי** — מנהל את היומן שלך, קורא מיילים, מזכיר דברים, עונה רק לך
> 2️⃣ **תמיכת לקוחות** — עונה ללקוחות בשם העסק, מטפל בשאלות נפוצות
> 3️⃣ **בוט ידע** — יודע לענות על נושא ספציפי (קורס, מוצר, תחום)
> 4️⃣ **בוט תיאומים** — קובע פגישות, מתאם לוחות זמנים

Save as `archetype`. Each archetype has sensible defaults; use them to skip questions later.

### Q2 — Name

> איך נקרא לו? שם פרטי, כינוי, מותג — מה שמתאים.

Save as `name`.

### Q3 — Audience

For `personal_assistant`: assume whitelist mode, ask:
> זה בוט אישי. מי מורשה לדבר איתו? (תן רשימת מספרים בפורמט +972..., אחד בשורה)

For `customer_support` / `knowledge_bot`: assume open mode, ask:
> כל אחד שיכתוב יקבל מענה?
> 1. כן, כולם
> 2. לא, רק רשימה מצומצמת

If whitelist: ask for numbers. If open: ask for a fallback when overloaded:
> מה הבוט יענה כשהוא לא מצליח לעזור (או מחוץ לשעות פעילות)?

### Q4 — Tone

> איך אתה רוצה שהבוט ידבר? אני אביא לך 3 דוגמאות, בחר את הסגנון הקרוב לרוחך:
>
> **א.** "היי 🙂 בטח, אני יכול לעזור עם זה. רגע, רק אבדוק..."
> **ב.** "שלום, אטפל בזה כעת. אעדכן בהמשך."
> **ג.** "היי! בכייייף 🔥 רק תן לי שנייה ✨"

Map to formality + warmth + emoji_usage. Then ask:

> רוצה לתת לי דוגמה של משפט אחד או שניים בסגנון שלך? זה עוזר לבוט להישמע כמוך, לא כמו רובוט.

Save 1-3 short Hebrew examples to `tone.voice_examples`.

### Q5 — In-scope / out-of-scope

> מה הבוט **אמור** לעשות? תן 2-4 דוגמאות.

Save to `scope.in`.

> מה הבוט **לא** צריך לעשות, גם אם יבקשו ממנו? (לדוגמה: לא לתת ייעוץ רפואי, לא לדבר על מתחרים, לא להבטיח מחיר)

Save to `scope.out`. Then:

> מה לענות כשמישהו מבקש משהו שמחוץ לתחום?

Save to `scope.out_of_scope_reply`.

### Q6 — Knowledge base

> יש משהו ספציפי שהבוט צריך לדעת? (מחירון, שעות פעילות, מדיניות, FAQ...)
>
> 1. לא, מספיק שהוא יהיה כללי וחכם
> 2. כן — אגיד לך עכשיו בכמה שורות
> 3. כן — יש לי קבצים (PDF, Word, אקסל) שנעלה

For 1: `knowledge_base.type = "none"`.
For 2: ask, save Hebrew text to `knowledge_base.content`, type `"inline"`.
For 3: ask user to drop files in a `./knowledge/` folder, type `"files"`. (Build stage will index them.)

### Q7 — Tools wishlist (light touch — full configuration in wa-connect)

> איזה יכולות תרצה שיהיו לבוט בהמשך? אפשר לבחור כמה. נחבר אותן אחת אחרי השנייה אחרי שהבוט הבסיסי יעבוד.
>
> ✅ יומן Google (קביעה, שינוי, ביטול פגישות)
> ✅ Gmail (קריאה, שליחה, סיכום)
> ✅ תזכורות מתוזמנות
> ✅ העברה לבן אדם (שאתה תקבל התראה במספר אחר)

Save selected to `tools_wishlist`. Build stage wires only `reminders` initially (fast first win); the rest are added in `wa-connect`.

### Q8 — Handoff to human

If `human_handoff` was selected:
> מה המספר שלך לקבלת התראות? (כשמישהו מבקש לדבר עם בן אדם)

Save to `handoff.owner_number`, set `handoff.enabled = true`.

> מה משפטי הטריגר שיגרמו להעברה? (למשל "תן לי לדבר עם מישהו", "אני רוצה נציג")

Save Hebrew strings to `handoff.trigger_phrases`. Default to `["דבר עם נציג", "להעביר לבן אדם", "אני רוצה איש תמיכה"]` if user doesn't have specifics.

### Q9 — LLM choice

For non-technical users, default and don't ask:
- `provider: "anthropic"`, `model: "claude-haiku-4-5"`, `max_history: 20`

Tell the user:
> בחרתי בשבילך את **Claude Haiku 4.5** — מהיר, חכם, וזול. תמיד אפשר לשדרג בהמשך.

Note: if the user explicitly said earlier they want OpenAI or Gemini, honor it (`gpt-5.4-nano` or `gemini-2.5-flash` as budget defaults).

## Save and confirm

Write `./spec.json` with everything collected. Then read it back to the user as a Hebrew summary:

> רגע, נסכם:
> - **שם הבוט**: {name}
> - **סוג**: {archetype}
> - **קהל**: {audience description}
> - **טון**: {tone summary}
> - **עושה**: {scope.in joined}
> - **לא עושה**: {scope.out joined}
> - **יכולות עתידיות**: {tools_wishlist joined}
>
> נראה לך טוב? אפשר לתקן כל פריט.

If user wants edits, modify `spec.json` accordingly.

## Update state

Update `.wa-state.json`:
```json
{
  "current_stage": "build",
  "completed_stages": ["setup", "characterize"],
  "bot_name": "<name>",
  "archetype": "<archetype>",
  "last_updated": "<ISO>"
}
```

Tell the user:

> מצוין. עכשיו אני בונה את הבוט עצמו על המחשב שלך — קוד שלם, מוכן להפעלה. ייקח לי דקה. אחרי זה נריץ אותו ונראה שהוא עונה.

Hand back to `/wa` which will load `wa-build`.
