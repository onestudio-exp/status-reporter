---
description: Capture the current session's delivered work as an append-only daily fragment for /status-export to consolidate later. Usage: /status-log [optional note]
argument-hint: [optional short note]
allowed-tools: Read, Write, Glob, Bash
---

# /status-log — capture this session's work (lightweight)

Records what **this specific session** delivered into a separate, append-only fragment file,
so multiple sessions in the same day accumulate without conflict. `/status-export` later
merges and polishes them into a single report.

- **Input:** this session's conversation context + `$ARGUMENTS` (an optional short note).
- **Output:** `docs/status/days/_fragments/<DATE>/<HHMM>.md` (relative to the current project root).
- **No PDF.** This command is lightweight and fast — capture only.

> **When to use it?** At the end of each work session (or whenever you've delivered something worth recording).
> Run it as many times a day as you like — each run is a separate file that never overwrites another and never
> conflicts with a parallel session. `/status-export` removes duplicates when it consolidates.

---

## Step 1 — Resolve date and time (Asia/Riyadh · no DST)

Riyadh time is a fixed UTC+3. Run the snippet matching the host OS:

```bash
# bash (macOS/Linux)
DATE=$(TZ=Asia/Riyadh date +%Y-%m-%d 2>/dev/null || date -u -d '+3 hours' +%Y-%m-%d)
TIME=$(TZ=Asia/Riyadh date +%H%M 2>/dev/null || date -u -d '+3 hours' +%H%M)
echo "DATE=$DATE TIME=$TIME"
```
```powershell
# PowerShell (Windows)
$riyadh = (Get-Date).ToUniversalTime().AddHours(3)
$DATE = $riyadh.ToString('yyyy-MM-dd'); $TIME = $riyadh.ToString('HHmm')
"DATE=$DATE TIME=$TIME"
```

Use this date. Do not ask the user for the date. Everything else in this command uses the
**Write tool** (the fragment) and the **Read tool** (config) — no further shell, so it's cross-platform.

---

## Step 2 — Summarize this session's work (source = your conversation)

**Language:** read `docs/status/config.json` if present and use its `language` field (`ar`/`en`) for the
fragment's written language. If no config exists, default to `ar` and do **not** prompt the user — this is a
lightweight command; project setup happens in `/status-export`.

Review this session's conversation from the start and extract **what was actually delivered** in it —
not what's planned, not what was done in a previous session.

Write **1–4 themes**, each a `## heading` followed by 2–5 bullets. Rules:

- **Value, not tooling** — write what the user/client can now do, not package or code names.
- **This session only** — if the session delivered nothing notable, say so plainly and don't invent themes.
- **Apply the scope filters** (same as Step 3 in `/status-export`) now, so the fragment comes out clean.
  The table depends on the language:

**Arabic (`ar`):**

| Banned | Replacement |
|---|---|
| "QA" / "جودة" in the technical sense | "مراجعة عمل · تصحيح الأخطاء إن وجدت" |
| "شارف على النهاية" / "تبقّى القليل" / "نهائي" | drop or neutralize the clause |
| claiming features not delivered this session | omit them |
| calendar dates inside the cards | leave them to the template — don't write them here |

**English (`en`):**

| Banned | Replacement |
|---|---|
| "QA" / "quality" in the technical sense | "work review · bug fixes where found" |
| "almost done" / "nearly finished" / "final" | drop or neutralize the clause |
| claiming features not delivered this session | omit them |
| calendar dates inside the cards | leave them to the template — don't write them here |

If the user passed `$ARGUMENTS`, treat it as a focus/instruction folded into the summary
(e.g. "focus on the users screen"), and also record it in the frontmatter `note` field.

---

## Step 3 — Write the fragment

Path: `docs/status/days/_fragments/<DATE>/<TIME>.md`
(Create the folder if needed. If a file with the same `<TIME>` already exists — rare — use `<TIME>-2.md`, etc.)

Format:

```yaml
---
date: <DATE>
logged_at: "<HH:MM>"
note: "<$ARGUMENTS if present, otherwise omit this line>"
---

## <theme heading>
- <bullet>
- <bullet>

## <theme heading>
- <bullet>
```

**Show the fragment content to the user before writing** (a few lines), then write it.

---

## Step 4 — One-line confirmation

```
Session logged → _fragments/<DATE>/<TIME>.md  (<theme count> themes · <bullet count> bullets)
Today's fragments: <number of files in today's folder>
Run /status-export to generate the consolidated daily report.
```

---

## Hard rules

1. **Append-only** — never overwrite, delete, or edit an existing fragment. Each run is a new file.
2. **No PDF, no template, no images** — this command is capture only; rendering is `/status-export`'s job.
3. **Don't touch the consolidated `<DATE>.md`** — consolidation happens in `/status-export`, not here.
4. **This session only** — don't pull from previous sessions or from older day files.
5. **Scope neutrality** — apply the filters above; when in doubt, flag the user instead of inventing.

## Stop conditions

- The session delivered nothing worth recording → tell the user and ask: record a brief note, or skip?
  Do not write an empty fragment.
