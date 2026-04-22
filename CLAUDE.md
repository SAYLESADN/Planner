# CLAUDE.md

Guidance for AI assistants (Claude Code and similar) working in this repo.

## What this is

**ADN Command Centre** — a personal task/priority manager built as a single-file static web app. It lives at `index.html` and runs straight from the filesystem or a static host (GitHub Pages). The target user is Sarah, who is running an HCM project with two hard dates baked into the UI: **UAT 2026-05-18** and **Go Live 2026-06-16**.

The app's tone is an intentional "sassy PA" voice. Greetings, toasts, and command responses all lean into it — preserve that when editing copy.

## Repo layout

```
.
├── CLAUDE.md     — this file
├── README.md     — placeholder (one word); don't expand unless asked
└── index.html    — the entire application (HTML + embedded <style> + <script>)
```

No `package.json`, no bundler, no tests, no framework. The only external asset is a Google Fonts CDN `<link>`. **Keep it that way** unless the user explicitly asks to add tooling.

## Running and "building"

There is no build step. Open `index.html` in a browser and it works. Reload to pick up edits. State is persisted to `localStorage` so it survives reloads — clear it via devtools if a fresh state is needed.

## Code map inside `index.html`

The `<script>` block is divided by `═══` banner comments. Jump to the right section by name:

| Section | Purpose |
|---|---|
| `STATE` | The single global `STATE` object; loaded from / saved to `localStorage['adn_cc_state']` via `save()`. |
| `DAY ROLLOVER` | `checkDayRollover()` — runs on every `render()`, logs yesterday's score, rolls undone active tasks, clears the calendar. |
| `ANALYST — SCORING` | `scoreTask(t)` — weighs urgency, impact, days waiting, avoidance (rolled count), effort. Returns 0–100. |
| `SASSY PA MESSAGES` | `getSassyGreeting()` — the top-of-view greeting generator. |
| `RENDER — TODAY / BACKLOG / WEEK / MILESTONES / STATS` | Five view renderers; switched by `STATE.currentView`. |
| `TASK ACTIONS` | `addTask`, `promoteTask`, `completeTask`, `demoteTask`, `cutTask`, subtask and milestone helpers. |
| `CALENDAR PASTE` | Parses pasted Outlook calendar text into `STATE.calendar`. |
| `COMMAND BAR` | `cmdReplan`, `cmdWhatNow`, `cmdStuck`, `cmdHonest`, `cmdForce` — each call `callClaude()` with a different system prompt. |
| `VIEW MANAGEMENT` | `showView`, `render`. |
| `MODALS` | Add-task, calendar-paste, sass, API-key, brain-dump overlays. |
| `APPLE PENCIL CANVAS` | Handwriting pad → Claude vision → tasks / notes / chat. |
| `CHAT ASSISTANT` + `ATTACHMENT HANDLING` | Per-task chat drawer. `readFileForApi()` converts uploads to API content blocks (PDFs → `document`, images → `image`, text/CSV/XLSX → inline text, truncated at 15k chars / 10 MB). |
| `SAVE / LOAD FILE` | Export / import the whole `STATE` as JSON. |
| `BRAIN FILE` | Optional JSON file that enriches system prompts with context about the user's work. |
| `CLAUDE API` | `_apiKey` + `callClaude(userMsg, systemExtra)` — the shared helper. |
| `EMAIL UPLOAD & PARSE` | `.eml` → Claude extraction → tasks (single file shows a review modal; multiple files bulk-import silently). |
| `INIT` | Calls `render()` at the bottom. |

## Data model

Every field a task can have (full shape — don't drop properties on round-trip):

```
{
  id, title, category, effort, deadline, notes, milestoneId, impact,
  created, active, done, doneDate, rolled, rolledDates,
  subtasks: [{ text, done }],
  attachments: [{ id, name, ext, size, addedAt, data }],
  emailId
}
```

Other collections on `STATE`:
- `milestones: [{ id, name, date }]`
- `calendar: [{ time, title }]`
- `completionLog: { 'YYYY-MM-DD': { completed, total } }`
- `emails: [{ id, filename, from, subject, date, body, parsed, uploadedAt }]`
- `currentView`, `lastDate`, `brainFile`

Saved state is read with a bare `JSON.parse` in a try/catch, so **always default-guard new fields** when you add them (`t.subtasks || []`). Existing stored state in the wild won't have them.

## Render contract

1. Any code that mutates `STATE` must call `save()` before `render()`.
2. `render()` always calls `checkDayRollover()` first, then dispatches to the active view, then updates `#topSub`.
3. View renderers replace `innerHTML` wholesale — do **not** hold DOM refs across renders; re-query by id inside handlers.

## Anthropic API integration

- Key lives in `localStorage['adn_cc_apikey']` and is read into module-scope `_apiKey`. Only the 🔑 modal sets it. **Don't** introduce a `.env` loader or commit a key.
- All requests use `'anthropic-dangerous-direct-browser-access': 'true'`.
- Current model id in the file: `claude-sonnet-4-20250514`. It appears at **three call sites** (search for `model:'claude-sonnet-4`). If you upgrade the model, update all three together.
- `callClaude(userMsg, systemExtra)` is the shared helper for text-only commands. Chat and pencil have their own `fetch` blocks because they pass multimodal content blocks.
- Hard-coded dates `'2026-05-18'` (UAT) and `'2026-06-16'` (Go Live) appear inside system prompts and the topbar so the model reports accurate day counts instead of hallucinating them. Keep them in sync with `STATE.milestones` if those dates ever change.

## Conventions when editing

- **Single-file discipline.** Everything stays in `index.html`. No splitting into modules, no build step, no framework.
- **Vanilla JS only.** No dependencies beyond the Google Fonts link.
- **Preserve tone.** The sassy copy in greetings, toasts, and command responses is part of the product — match the voice when adding strings.
- **Respect the 3-active-task cap.** `promoteTask()` enforces it deliberately; don't relax it.
- **IDs** use `uuid()` (prefix `t` + timestamp + random). Don't hand-roll.
- After any STATE mutation: `save(); render();`.

## Known quirks / gotchas

- **CSS custom-property references use the wrong dash across most of the stylesheet.** 91 references use an en-dash `–` (U+2013) — e.g. `var(–navy)` — and 64 use the correct `--`. The `:root` declarations themselves are all correct `--`. The broken references silently fall back to defaults. Before doing a mass find/replace, confirm with the user: the fix will visibly change the app's colours everywhere at once, so it's a deliberate call, not a drive-by cleanup.
- **Day rollover is lazy.** It only runs inside `render()`. If the tab is left open past midnight, rollover fires on the next interaction, not on a timer.
- **`.eml` parsing is naïve.** Headers are parsed line-by-line, MIME multipart is ignored, HTML is stripped with a regex, body is truncated to 3000 chars before being sent to Claude. Don't assume full fidelity.
- **Attachments are stored in `localStorage` as base64.** Large PDFs or images will blow the quota quickly. The 10 MB per-file cap is enforced in `readFileForApi()`; the overall cap is whatever the browser gives localStorage (~5 MB on many browsers — yes, smaller than a single attachment). Expect quota errors on multi-file uploads; `save()` swallows them silently via its try/catch.
- **`anthropic-dangerous-direct-browser-access`** is required because we call the Anthropic API straight from the browser. This exposes the key to anyone with the user's device / browser storage. Accepted trade-off for a personal tool; don't deploy as multi-user.

## What NOT to do

- Don't add `package.json`, a bundler, TypeScript, or a framework.
- Don't create new top-level files unless asked.
- Don't refactor `index.html` into modules.
- Don't silently "fix" the en-dash CSS issue — surface it first.
- Don't strip the sassy tone from UI strings.
- Don't commit API keys or a `.env`.

## Git workflow

- Per this task's handoff, development happens on branch **`claude/add-claude-documentation-VPaAg`**. Confirm with the user before switching branches.
- Commit directly, push with `git push -u origin <branch>`.
- GitHub MCP tools are restricted to `saylesadn/planner` — don't attempt other repos.
- Do **not** open a PR unless the user explicitly asks.
