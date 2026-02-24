# CLAUDE.md — AI-verktyg / CV-Byggaren

This file describes the codebase for AI assistants working in this repository.

---

## Project Overview

**CV-Byggaren** is a Swedish-language, single-page web application that lets users build professional CVs/resumes. It offers 50 templates (10 layouts × 5 color themes), a guided 5-step form builder, AI-powered text generation via the Claude API, and browser-native PDF export.

The entire application lives in one file: `index.html` (≈951 lines). There is no build system, no package manager, no server, and no external dependencies beyond Google Fonts and the Anthropic API.

---

## Repository Structure

```
AI-verktyg/
├── index.html      # Complete application — HTML + CSS + JS in one file
└── README.md       # Minimal placeholder ("# AI")
```

No other source files, configuration files, test files, or build scripts exist.

---

## Technology Stack

| Layer      | Technology                                      |
|------------|-------------------------------------------------|
| Markup     | HTML5 (`lang="sv"`)                             |
| Styling    | Vanilla CSS3 (Grid, Flexbox, custom properties) |
| Logic      | Vanilla JavaScript ES6+ (no frameworks)         |
| AI         | Anthropic Claude API (`claude-sonnet-4-20250514`) |
| Fonts      | Google Fonts (loaded via CDN)                   |
| PDF export | Browser print dialog (`window.print()`)         |

**No npm, no bundler, no TypeScript, no framework.** Opening `index.html` directly in a browser is the only setup required.

---

## Architecture

### Page Model

The app has two top-level "pages" toggled with CSS `.active`:

| Page ID        | Content                          |
|----------------|----------------------------------|
| `#page-landing` | Marketing landing page           |
| `#page-app`    | Gallery + Builder + Preview      |

```js
function showPage(p) {
  // removes .active from all .page elements, adds it to #page-{p}
  // lazy-initialises the gallery on first visit to 'app'
}
```

### App Sections (inside `#page-app`)

1. **`#gallery`** — Template picker with category filters and live thumbnail previews
2. **`#builder`** — 5-step form (hidden until a template is selected)
3. **`#preview`** — Rendered CV with template switcher and download button (hidden until CV is generated)

---

## Template System

### Data

```js
const COLORS = ['navy', 'forest', 'charcoal', 'burgundy', 'slate'];

const LAYOUTS = [
  { id: 'kronos', name: 'Kronos', cat: ['klassisk', 'kontor'],   desc: 'Executive klassiker' },
  { id: 'sigma',  name: 'Sigma',  cat: ['modern', 'tech'],       desc: 'Sidofält modern' },
  { id: 'aurum',  name: 'Aurum',  cat: ['klassisk', 'kreativ'],  desc: 'Elegant serif' },
  { id: 'void',   name: 'Void',   cat: ['modern', 'kreativ'],    desc: 'Ultra minimal' },
  { id: 'grid',   name: 'Grid',   cat: ['klassisk', 'sjukvard'], desc: 'Symmetrisk tvåkolumn' },
  { id: 'pulse',  name: 'Pulse',  cat: ['tech', 'modern'],       desc: 'Tech-fokuserad' },
  { id: 'flora',  name: 'Flora',  cat: ['sjukvard', 'klassisk'], desc: 'Varmt & personligt' },
  { id: 'motion', name: 'Motion', cat: ['kreativ', 'modern'],    desc: 'Kreativ mörk' },
  { id: 'strata', name: 'Strata', cat: ['modern', 'tech'],       desc: 'Bandhuvud' },
  { id: 'apex',   name: 'Apex',   cat: ['klassisk', 'sjukvard'], desc: 'Accentlinje' },
];

// 50 templates are generated: each layout × each color
const TEMPLATES = [];
LAYOUTS.forEach(lay => COLORS.forEach(col => TEMPLATES.push({
  id: `${lay.id}-${col}`,
  layout: lay.id,
  color: col,
  name: `${lay.name} ${COLOR_LABELS[col]}`,
  cat: lay.cat,
  desc: lay.desc
})));
```

Template IDs follow the pattern `{layout}-{color}`, e.g. `kronos-navy`, `sigma-forest`.

### Color CSS Variables

Each template uses five CSS custom properties scoped to `[data-color="{color}"]`:

| Variable | Role                        |
|----------|-----------------------------|
| `--c1`   | Primary / darkest            |
| `--c2`   | Secondary                   |
| `--c3`   | Accent / muted text          |
| `--c4`   | Light background tint        |
| `--c5`   | Very light background tint   |

These are injected both in the app's `<style>` block and duplicated inline in the download HTML (`dlCV()`).

### Rendering Pipeline

```
renderCV(data, templateId)
  └─ renderTemplate(layout, color, data)
       └─ renderKronos / renderSigma / renderAurum / renderVoid /
          renderGrid / renderPulse / renderFlora / renderMotion /
          renderStrata / renderApex
            └─ returns HTML string injected into #cv-out
```

Each `renderXxx(d, co, sk, c)` function receives:
- `d` — full CV data object
- `co` — contact info array (pre-filtered from `[email, tel, stad, li, web]`)
- `sk` — skills array
- `c` — color string (e.g. `'navy'`)

Two shared helpers reduce repetition across renderers:

```js
function jHtml(j, prefix)  // renders one job entry
function uHtml(u, prefix)  // renders one education entry
```

---

## CV Data Model

All user input is collected via `getData()` into this object:

```js
{
  fn:      string,   // First name     (field: #p-fn)
  en:      string,   // Last name      (field: #p-en)
  titel:   string,   // Job title      (field: #p-ti)
  email:   string,   // Email          (field: #p-em)
  tel:     string,   // Phone          (field: #p-tel)
  stad:    string,   // City           (field: #p-stad)
  li:      string,   // LinkedIn URL   (field: #p-li)
  web:     string,   // Website        (field: #p-web)
  summary: string,   // Profile text   (field: #s-fin or #s-raw)
  skills:  string[], // Comma-split from #k-sk
  ovrigt:  string,   // Other info     (field: #k-ov)
  jobb: [            // Work experience (repeatable)
    { pos, org, start, slut, stad, desc }
  ],
  utb: [             // Education (repeatable)
    { prog, skola, start, slut, stad, betyg }
  ],
  sprak: [           // Languages (repeatable)
    { n, l }         // name, level
  ]
}
```

Helper `v(id)` reads and trims an input's value: `document.getElementById(id)?.value.trim() || ''`.

---

## State Variables

All global state is declared at the top of the `<script>` block:

```js
let selTpl  = null;  // Selected template ID (string like 'kronos-navy')
let step    = 1;     // Current builder step (1–5)
let jCount  = 0;     // Number of job entries added (ever; never decremented)
let uCount  = 0;     // Number of education entries added
let lCount  = 0;     // Number of language entries added
let lastData = {};   // Last getData() result, used for template switching in preview
```

> **Important:** `jCount`, `uCount`, `lCount` are monotonically increasing counters.
> When a repeatable block is removed via "Ta bort", its DOM node is deleted but the counter stays the same. `getData()` skips missing blocks with `document.getElementById('jb'+i)` null checks.

---

## Form Builder — 5 Steps

| Step | Panel ID  | Content                        |
|------|-----------|--------------------------------|
| 1    | `#pan-1`  | Personal info (name, title, contact) |
| 2    | `#pan-2`  | Professional summary + AI generation |
| 3    | `#pan-3`  | Work experience (repeatable)   |
| 4    | `#pan-4`  | Education (repeatable)         |
| 5    | `#pan-5`  | Skills, languages, other info  |

Navigation:
- `goStep(n)` — jump to step `n`
- `stepFwd()` — next step; calls `genCV()` on step 5
- `stepBack()` — previous step

---

## AI Integration

The app calls the Anthropic Messages API directly from the browser (no backend proxy):

```js
async function claude(prompt) {
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 600,
      system: "Du är expert på svenska CV-texter. Svara ENBART med den färdiga texten – ingen rubrik, inga förklaringar.",
      messages: [{ role: "user", content: prompt }]
    })
  });
  const d = await r.json();
  if (d.error) throw new Error(d.error.message);
  return d.content?.[0]?.text?.trim() || '';
}
```

**The API key is entered by the user at runtime** via a visible input field in the UI — it is never hardcoded and never stored server-side.

Three AI-powered actions exist:

| Function    | Trigger                     | Output field      |
|-------------|-----------------------------|-------------------|
| `genSum()`  | "Generera med AI" button    | `#s-fin`          |
| `genJob(n)` | "Förbättra med AI" per job  | `#j{n}f`          |
| Auto-sum    | Inside `genCV()` when `summary` is empty | `#s-fin` |

All prompts are in Swedish. Responses are plain text with no headings or explanations (enforced by the system prompt).

---

## PDF Export

`dlCV()` opens a new browser window, writes a self-contained HTML document (including all CV CSS and Google Fonts), then calls `window.print()` after a 900 ms delay to let fonts load. The user saves via the browser's print-to-PDF dialog.

---

## UI Conventions

### CSS Design Tokens (dark app theme)

```css
:root {
  --bg:       #0A0A08;   /* Page background */
  --s1:       #141412;   /* Surface 1 */
  --s2:       #1E1E1B;   /* Surface 2 */
  --border:   #2E2E2B;   /* Borders */
  --text:     #F2EFE8;   /* Primary text */
  --muted:    #7A7772;   /* Secondary text */
  --acc:      #E8C547;   /* Gold accent */
  --acc-dim:  #2E2800;   /* Accent background */
  --green:    #4ADE80;   /* Success */
  --red:      #FF6B6B;   /* Error */
}
```

### Font Stack

| Role         | Font                        |
|--------------|-----------------------------|
| Headings     | Syne (800 weight)           |
| Serif accent | Instrument Serif (italic)   |
| Monospace    | JetBrains Mono              |
| Body         | DM Sans                     |
| Premium CV   | Cormorant Garamond, Space Grotesk |

### Feedback Patterns

- **Toast** — `toast(msg)`: bottom-right notification, auto-dismisses after 3.2 s (`#toast`)
- **Overlay** — `showOv(text)` / `hideOv()`: full-screen spinner during AI calls (`#ov`, `#ov-t`)

### Responsive Breakpoints

| Breakpoint    | Change                                   |
|---------------|------------------------------------------|
| `≤ 1100px`    | Grid adjustments                         |
| `≤ 900px`     | Two-column → single column layouts       |
| `≤ 560px`     | Template grid: 5 → 3 → 2 columns        |

---

## Development Workflow

### Running the App

No installation needed. Open `index.html` in any modern browser:

```bash
# Option 1: Direct file open
open index.html

# Option 2: Simple dev server (avoids CORS on fetch)
python3 -m http.server 8080
# then visit http://localhost:8080
```

A dev server is recommended because the AI features use `fetch()` to the Anthropic API; some browsers block this from `file://` origins.

### Making Changes

All HTML, CSS, and JS are in `index.html`. The file is structured with clear comment delimiters:

```
/* ── RESET & BASE ── */
/* ══ LANDING PAGE ══ */
/* ══ APP PAGE ══ */
// ── PAGE SWITCHING ──
// ── TEMPLATES ──
// ── GALLERY ──
// ── STEPS ──
// ── REPEATABLE FIELDS ──
// ── AI ──
// ── DATA ──
// ── GENERATE & PREVIEW ──
// ── DOWNLOAD ──
// ── UTILS ──
```

Use these markers to navigate. There is no minification step — the file is already hand-authored and readable.

### Adding a New Template Layout

1. Add an entry to the `LAYOUTS` array with a unique `id`, display `name`, filter `cat` array, and `desc`.
2. Write a `renderXxx(d, co, sk, c)` function that returns an HTML string for the CV.
3. Add a `case 'xxx':` branch in `renderTemplate()`.
4. Add the layout's CSS styles inside both the main `<style>` block (for the preview) and inside the `dlCV()` inline styles string (for the downloaded PDF).

### Adding a New Color Theme

1. Add the color key to the `COLORS` array.
2. Add a label to `COLOR_LABELS`.
3. Add a `[data-color="{key}"]` CSS rule with `--c1` through `--c5` in both the main `<style>` block and inside `dlCV()`.

---

## No Tests, No CI

There are no automated tests and no CI/CD configuration. Manual browser testing is the only verification method. When making changes, verify by opening `index.html` and exercising the full user flow: landing → gallery → builder (all 5 steps) → generate CV → switch templates → download.

---

## Key Constraints and Gotchas

- **No backend.** The Anthropic API key is provided by the user each session. Do not add any server-side code.
- **Single file.** Keep everything in `index.html`. Do not split into separate CSS/JS files unless explicitly asked.
- **Swedish language.** All user-visible text, labels, placeholders, and AI prompts are in Swedish. Maintain this.
- **Counter monotonicity.** `jCount`, `uCount`, `lCount` never decrease. `getData()` skips removed DOM nodes — preserve this pattern when modifying repeatable fields.
- **Template CSS duplication.** Template styles exist in two places: the app `<style>` block and the inline `<style>` string inside `dlCV()`. Any template CSS change must be applied in both locations.
- **Font loading delay in PDF.** The 900 ms `setTimeout` before `w.print()` in `dlCV()` exists to let Google Fonts load. Do not reduce this value.
- **No localStorage / persistence.** User data is not saved between sessions. This is by design.
