---
description: Generate a professional daily status report as a PDF from a fixed HTML template. Usage: /status-export [YYYY-MM-DD]
argument-hint: [YYYY-MM-DD] | (empty for today)
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, TodoWrite, mcp__playwright__browser_navigate, mcp__playwright__browser_run_code_unsafe, mcp__playwright__browser_close, mcp__plugin_playwright_playwright__browser_navigate, mcp__plugin_playwright_playwright__browser_run_code_unsafe, mcp__plugin_playwright_playwright__browser_close
---

# /status-export — professional daily status report (PDF)

Produces a single PDF report for a given day, using one fixed template you must not deviate from.
**Input:** the daily Markdown file `docs/status/days/<DATE>.md` (+ session fragments).
**Output:** a PDF at `docs/status/reports/<DATE>-status.pdf`.

All paths are relative to the current project root (the working directory). Never assume a hard-coded
absolute path tied to any one project.

## Platform & tooling portability (read first)

This command runs on Windows, macOS, and Linux. To stay portable:

- **Prefer Claude's native tools over shell** for file work: use **Read** (config, template, day file),
  **Write** (day file, HTML), and **Glob** (list fragments). These need no shell and work everywhere.
- Where a genuine shell step is unavoidable (date, PDF render, archiving), snippets are given for **both**
  **bash** (macOS/Linux) and **PowerShell** (Windows). Run the one matching the host OS.
- **PDF rendering uses a headless Chromium-family browser via its CLI** (Chrome / Edge / Chromium / Brave).
  This needs no MCP server and is the portable default. A Playwright MCP, if present, is only an optional fallback.

---

## Inputs

- `$ARGUMENTS` = optional `YYYY-MM-DD`. If empty, use today in the report timezone (default Asia/Riyadh).

## Expected output

- `docs/status/reports/<DATE>-status.pdf`
- A final summary for the user: path, file size, page count.

---

## Step 0 — Configuration (project name + language)

Read `docs/status/config.json` with the **Read tool** (no shell). Fields: `project_name`, `subtitle`, `language` (`ar` or `en`).

- **If it exists** → use its values.
- **If it doesn't (first run)** → ask the user **once** via `AskUserQuestion` (two batched questions):
  the project name, and the default language (Arabic/English). They may also provide a `subtitle`;
  otherwise derive a neutral one in the chosen language. Then **Write** `docs/status/config.json`:
  ```json
  { "project_name": "<NAME>", "subtitle": "<SUBTITLE>", "language": "<ar|en>" }
  ```
  This is a one-time file; later runs read it without asking. The user edits it manually anytime to
  rename the project or switch language.

**The resolved language (`<LANG>`) governs the rest of the command**: the language the content is written in
(Steps 2–3), the filter table (Step 3), and the template chrome + direction (Step 4). Per-language string set:

| token | `ar` | `en` |
|---|---|---|
| `{{LANG}}` | `ar` | `en` |
| `{{DIR}}` | `rtl` | `ltr` |
| `{{DOC_TITLE}}` | `<project_name> — تقرير حالة #<DAY_NUMBER>` | `<project_name> — Status Report #<DAY_NUMBER>` |
| `{{BRAND_NAME}}` | `<project_name>` | `<project_name>` |
| `{{BRAND_SUBTITLE}}` | `<subtitle>` | `<subtitle>` |
| `{{DAY_LABEL}}` | `تقرير حالة · يوم #<DAY_NUMBER>` | `Status report · Day #<DAY_NUMBER>` |
| `{{CONTENT_TAG}}` | `إنجازات اليوم` | `Today's work` |
| `{{BACK_NOTE}}` | `أُعِدّ للمشاركة مع الفريق والإدارة` | `Prepared for the team and management` |

Format `{{DATE}}` for the chosen language (e.g. `2026-06-09` stays as-is, or a readable locale form).

---

## Step 1 — Resolve date and day number

If `$ARGUMENTS` holds a `YYYY-MM-DD`, use it. Otherwise compute today in the report timezone
(Riyadh = fixed UTC+3, no DST):

```bash
# bash (macOS/Linux)
DATE=$(TZ=Asia/Riyadh date +%Y-%m-%d 2>/dev/null || date -u -d '+3 hours' +%Y-%m-%d)
echo "DATE=$DATE"
```
```powershell
# PowerShell (Windows)
$DATE = (Get-Date).ToUniversalTime().AddHours(3).ToString('yyyy-MM-dd')
"DATE=$DATE"
```

Resolve `<DAY_NUMBER>`:
- If `docs/status/days/<DATE>.md` exists with a `day_number` in frontmatter → use it.
- Otherwise → count existing `docs/status/days/*.md` and add 1. **The Glob tool matches recursively**, so
  **exclude any path under `_fragments/`** — count only direct day files in `days/`.

Announce the result on one line, then continue.

---

## Step 2 — Consolidate fragments, then read/create the day file

List the session fragments with the **Glob tool**: `docs/status/days/_fragments/<DATE>/*.md`.
**The Glob tool matches recursively**, so **exclude any path under `_consumed/`** — those are already-archived
fragments and must not be re-read (re-reading them double-counts work on a rerun). Also check the consolidated
day file `docs/status/days/<DATE>.md` with **Read**.

> **Language**: everything written in this step — the merged content, `title`, `summary`, card headings and
> bullets — is in the `<LANG>` resolved in Step 0. If any fragments were written in a different language
> (logged before the language changed), translate them to the target language while merging.

Handle by case:

### Case (a) — unconsumed fragments exist → merge and polish (primary path)

1. **Read** all `_fragments/<DATE>/*.md` files, sorted chronologically by name.
2. **Read** the consolidated `<DATE>.md` if it exists — previously recorded work; don't drop it.
3. **Merge and polish** everything into a final set of themes:
   - **De-duplicate**: the same task in two sessions → one card combining the bullets.
   - **Order logically**, not chronologically (related themes adjacent), 3–5 bullets per card.
   - Apply the scope filters (Step 3) to the merged text.
4. **Auto-draft** `title` (one line) and `summary` (one or two sentences) from the union of themes.
5. Show the full draft (frontmatter + themes) to the user and let them tweak it before writing
   (apply their edits; don't use `AskUserQuestion` unless the title/summary is genuinely ambiguous).
6. **Write** the result to `docs/status/days/<DATE>.md` in the standard format below.
7. **Archive** the consumed fragments so they aren't double-counted on rerun:
   ```bash
   # bash
   mkdir -p "docs/status/days/_fragments/$DATE/_consumed"
   mv docs/status/days/_fragments/$DATE/*.md "docs/status/days/_fragments/$DATE/_consumed/" 2>/dev/null || true
   ```
   ```powershell
   # PowerShell
   $c = "docs/status/days/_fragments/$DATE/_consumed"; New-Item -ItemType Directory -Force -Path $c | Out-Null
   Get-ChildItem "docs/status/days/_fragments/$DATE" -Filter *.md -File | Move-Item -Destination $c -Force
   ```

### Case (b) — no fragments, but `<DATE>.md` exists → legacy behavior

- **Read** it fully (YAML frontmatter + Markdown body).
- Verify the fields exist: `day_number`, `title`, `summary`. If something is missing → request it via `AskUserQuestion`.

### Case (c) — no fragments and no day file → manual creation (legacy behavior)

Use `AskUserQuestion` (one question if possible) to request the day's title, a summary, and the
achievements list (the user writes it as multi-line Markdown).

> Suggest instead that the user run `/status-log` in future sessions so it consolidates automatically.

### Day file format (cases a and c)

```yaml
---
day_number: <N>
title: "<TITLE>"
summary: "<SUMMARY>"
screenshots_dir: "assets/screenshots/<DATE>"  # optional
---

<MARKDOWN BODY>
```

Show the file content to the user before writing it.

---

## Step 3 — Scope discipline (mandatory)

Before converting content to HTML, apply the scope filters per `<LANG>`:

**Arabic (`ar`):**

| Banned | Replacement |
|---|---|
| "QA" / "جودة" in the technical sense | "مراجعة عمل · تصحيح الأخطاء إن وجدت" |
| "شارف على النهاية" / "تبقّى القليل" / "نهائي" | drop the clause or reword it neutrally |
| claims about another day's features | flag the user, confirm before including |
| calendar dates inside the day cards | allow only in the footer |

**English (`en`):**

| Banned | Replacement |
|---|---|
| "QA" / "quality" in the technical sense | "work review · bug fixes where found" |
| "almost done" / "nearly finished" / "final" | drop the clause or reword it neutrally |
| claims about another day's features | flag the user, confirm before including |
| calendar dates inside the day cards | allow only in the footer |

If you find a conflict, tell the user and wait for a decision before continuing.

---

## Step 4 — Convert to HTML

1. **Resolve the template** with the **Read tool**, in this order (project takes precedence over bundled):
   - If `docs/status/_template.html` exists in the project → use it (lets each project keep its own identity).
   - Otherwise → use the bundled template: `${CLAUDE_PLUGIN_ROOT}/assets/_template.html`.
2. Convert the Markdown body to HTML — each `## heading` becomes a `.feature` card with an `<h3>` and a `<ul>` below.
3. Replace the **content** placeholders: `{{DATE}}`, `{{DAY_NUMBER}}`, `{{TITLE}}`, `{{SUMMARY}}`,
   `{{FEATURES_HTML}}`, `{{SCREENSHOTS_HTML}}`.
4. Replace the **identity & language** placeholders from the Step 0 table per `<LANG>` and `config`:
   `{{LANG}}` · `{{DIR}}` · `{{DOC_TITLE}}` · `{{BRAND_NAME}}` · `{{BRAND_SUBTITLE}}` · `{{DAY_LABEL}}` · `{{CONTENT_TAG}}` · `{{BACK_NOTE}}`.
   (If a project-local template lacks these tokens — a fixed-identity template — ignore what's absent; replace only what's present.)
5. **Write** the resulting HTML to `docs/status/.tmp/<DATE>/index.html`.

---

## Step 5 — Screenshots (optional)

If a folder `docs/status/assets/screenshots/<DATE>/` exists and contains `.png` files:

1. Copy the images into `docs/status/.tmp/<DATE>/` (next to `index.html`).
2. For each image, append a `<section class="page shot">` to `{{SCREENSHOTS_HTML}}` containing:
   - A title (the filename without leading number and extension, readable in the report language if possible)
   - The image with a **relative** `src` (e.g. `src="screen-1.png"`) — the CLI renderer loads the page over
     `file://`, so relative paths resolve. (Only the Playwright-MCP fallback below needs base64 inlining instead.)
   - A screen number (e.g. "Screen 2 / 5")

If the folder doesn't exist → skip this step. The report contains no images.

---

## Step 6 — Render the PDF (headless Chromium CLI — portable, no MCP)

Resolve the project root and absolute paths, then find a Chromium-family browser:

```bash
# bash (macOS/Linux)
ROOT=$(pwd); mkdir -p "docs/status/reports"
PDF="$ROOT/docs/status/reports/$DATE-status.pdf"
HTML="file://$ROOT/docs/status/.tmp/$DATE/index.html"
BROWSER=""
for c in google-chrome google-chrome-stable chromium chromium-browser microsoft-edge brave-browser; do
  command -v "$c" >/dev/null 2>&1 && { BROWSER="$c"; break; }
done
if [ -z "$BROWSER" ]; then for p in \
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  "/Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge" \
  "/Applications/Chromium.app/Contents/MacOS/Chromium"; do
  [ -x "$p" ] && { BROWSER="$p"; break; }; done; fi
echo "BROWSER=$BROWSER"; echo "PDF=$PDF"; echo "HTML=$HTML"
```
```powershell
# PowerShell (Windows)
$ROOT = (Get-Location).Path -replace '\\','/'; New-Item -ItemType Directory -Force -Path "docs/status/reports" | Out-Null
$PDF  = "$ROOT/docs/status/reports/$DATE-status.pdf"
$HTML = "file:///$ROOT/docs/status/.tmp/$DATE/index.html"
$BROWSER = @(
  "$env:ProgramFiles\Google\Chrome\Application\chrome.exe",
  "${env:ProgramFiles(x86)}\Google\Chrome\Application\chrome.exe",
  "$env:ProgramFiles\Microsoft\Edge\Application\msedge.exe",
  "${env:ProgramFiles(x86)}\Microsoft\Edge\Application\msedge.exe"
) | Where-Object { Test-Path $_ } | Select-Object -First 1
"BROWSER=$BROWSER"; "PDF=$PDF"; "HTML=$HTML"
```

Then render (the `@page { size: A4 landscape; margin: 0 }` in the template drives page size and margins):

```bash
# bash
"$BROWSER" --headless=new --disable-gpu --no-pdf-header-footer --print-to-pdf="$PDF" "$HTML"
```
```powershell
# PowerShell
& $BROWSER --headless=new --disable-gpu --no-pdf-header-footer --print-to-pdf="$PDF" "$HTML"
```

Then clean up the temp folder (bash `rm -rf "docs/status/.tmp/$DATE"` / PowerShell `Remove-Item -Recurse -Force "docs/status/.tmp/$DATE"`).

### Fallback — Playwright MCP (only if no Chromium browser is found)

If browser detection returns empty AND a Playwright MCP is available (tool name varies by install, e.g.
`mcp__playwright__*` or `mcp__plugin_playwright_playwright__*`): the MCP blocks `file://` and its snippet
sandbox has no `fs`, so navigate to `about:blank`, then `..._run_code_unsafe` with the **full generated HTML
inlined** into `page.setContent(html)` then `page.pdf({ path, format:'A4', landscape:true, printBackground:true,
margin:{top:'0',bottom:'0',left:'0',right:'0'}, preferCSSPageSize:true })`. In this path, screenshots must be
**base64 data-URI inlined** (no base URL). Close the browser when done.

---

## Step 7 — Final report to the user

```
Done: docs/status/reports/<DATE>-status.pdf
Size: <SIZE>
Pages: <N>
Day: #<DAY_NUMBER> — <TITLE>
```

Read the size with bash `du -h <file>` / PowerShell `"{0:N0} KB" -f ((Get-Item $f).Length/1KB)`.
If the PDF is missing, report that generation failed (no browser found, or render error).

---

## Hard rules

1. **The template is the single visual source of truth** — don't inject per-run custom CSS (projects customize only via `docs/status/_template.html`).
2. **The date appears only in the cover/footer** — never inside the achievement cards.
3. **Day numbering is cumulative** — never "reset to 1".
4. **Never include images the user didn't place in `assets/screenshots/<DATE>/`** — never auto-capture screenshots.
5. **Clean `.tmp/`** — the temp folder must be removed at the end, even on failure.
6. **Don't modify any file outside `docs/status/`** — this command is isolated to the status folder.

## Stop conditions

- No Chromium-family browser found **and** no Playwright MCP available → stop and tell the user to install Chrome/Edge/Chromium.
- No day file and the user didn't answer `AskUserQuestion` → stop.
- Step 3 scope discipline found a conflict the user hasn't authorized → stop.
- A PDF for an earlier date already exists and you're regenerating it → ask first via `AskUserQuestion`.
