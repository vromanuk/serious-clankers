# HTML visual kit (explain-topic)

Load when building the explanation page. Patterns are generic; adapt to the topic.

## Goals

- Intuition from **structure, flow, and familiar example data** — all visible at once.
- A **small set of diagram families** reused across the page.
- Self-contained file: no CDN, no packages.
- **No Play/step players by default** — static HTML/CSS figures.
- Examples on figures: **relevant / simple / teaching / familiar** (named roles
  like Checkout + Email, not bare A/B + `allocate`/`free` unless that is the topic).

## Suggested design tokens

```css
:root {
  --bg: #f6f4ef;
  --paper: #fffdf8;
  --ink: #1c1b19;
  --muted: #5c574e;
  --line: #d9d2c3;
  --accent: #1f5f8b;
  --accent-soft: #e6f1f8;
  --good: #1f6b3a;
  --good-soft: #e5f5ea;
  --warn: #8a5a00;
  --warn-soft: #fff3d6;
  --bad: #8b1f2d;
  --bad-soft: #fde8eb;
  --purple: #5b3d8a;
  --purple-soft: #f0e9f8;
  --shadow: 0 10px 30px rgba(28, 27, 25, 0.08);
  --radius: 14px;
  --mono: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  --serif: "Iowan Old Style", "Palatino Linotype", Palatino, Georgia, serif;
  --sans: "Avenir Next", "Segoe UI", system-ui, sans-serif;
}
```

Body serif for prose; sans for UI labels; mono for code. Soft paper background.

## Page skeleton

```text
header      title, spine summary
nav.toc     links to required sections
#background
#why
#intuition   (figures here)
#how         (more figures as needed)
#compare
#quiz
#study-cards  (optional)
footer
```

## Callouts

| Role | Use |
|---|---|
| definition | term on first use |
| why | need / constraint |
| trap | common misconception |
| edge | important edge case |
| remember | sticky one-liner |

## Diagram families (static HTML/CSS)

### Before / after

```html
<div class="compare">
  <div class="panel bad">
    <h4>Without / before</h4>
    …
  </div>
  <div class="panel good">
    <h4>With / after</h4>
    …
  </div>
</div>
```

### Flow with example data

Horizontal or vertical **cards** with plain titles; put sample values on edges
or under boxes (`user_id=7`, `File open`, `job: Vec<u8>`).

### Whole picture, then parts

For lifecycles and protocols, **first** a single overview figure (before/after
vs the usual design, or one scope that contains the full story). **Then** a
table or numbered steps that name phases *inside* that picture. Never only a
1→2→3 strip without the overview — readers see fragments, not the model.

### Numbered steps (all visible) — after the overview

```html
<ol class="steps">
  <li><strong>Open</strong> — … <code>…</code></li>
  <li><strong>Use</strong> — …</li>
  <li><strong>Drop</strong> — …</li>
</ol>
```

Or a row of always-bright cards `1 → 2 → 3` — not dimmed, not gated on Play.
Each step’s caption should say how it sits in the big picture.

### Tables

Toy inputs → outputs; who owns what after a call.

### Do not (default)

- ASCII art as the main figure  
- Play / Next / chip timeline players  
- SVG node graphs that need animation to mean anything  
- Figures with no example data when data is the point  

## Code blocks

```css
pre, pre code {
  white-space: pre-wrap;
  font-family: var(--mono);
}
```

Use `<pre><code>…</code></pre>`. Escape `<`. Scan before save.

## Quiz UI

Five questions; shuffle options; feedback after click; offline JS only.

## Study cards UI (optional — separate from figures)

Static **Q: / A:** list — both sides always visible, easy to select and copy.
No flip, no click-to-reveal, no hide-answer.

**Content priority:** intuition first — toy values and one-step walkthroughs
(“How can `"hello"` be represented internally?”), not glossary-only cards.
See `SKILL.md` § Study cards for the full pattern table.

```html
<article class="study-item">
  <div class="study-tag">intuition</div>
  <p class="study-q"><span class="qa-label">Q:</span> How can the string "hello" be implemented internally?</p>
  <div class="study-a">
    <span class="qa-label">A:</span>
    <span class="study-a-body">Lead sentence.

- Bullet one

- Bullet two

Closing line if needed.</span>
  </div>
</article>
```

Suggested CSS ideas (adapt tokens):

- Card: paper background, left accent border, normal `user-select: text`
- Q: sans, slightly bold
- A: soft accent background so answer is visually distinct but fully readable
- **Answer body: `white-space: pre-wrap`** so lead + bullets stay readable and copy-friendly
- Labels `Q:` / `A:` in accent / good colors
- Answers: lead sentence, then `-` bullets for lists of designs/steps/downsides — not one dense paragraph

Rules:

- **No** flip / 3D / “click to see answer” unless the user asks  
- **No** labeled slots (SCENE / REMEMBER / …)  
- **No** diagram shells inside cards  
- **Copy button on every card** — clipboard plain text:

  ```text
  Q: <question>

  A:
  <answer with newlines and - bullets>
  ```

  (`clipboard.writeText` + textarea fallback; temporary “Copied” label)  
- Optional topic filter buttons are fine (show/hide whole cards only)  
- Visuals stay in the main article sections only  
- Prefer at least one **toy-value / one-operation** card when the page compares layouts or lifecycles

## Quality bar

- [ ] Teaching order: background → why → intuition (with figures) → how → quiz  
- [ ] Intuition anchored on a **real familiar instance of the topic** (CSV row,
      real path, stack of plates), **not an invented metaphor** — metaphor only
      as last resort when no familiar instance exists (SKILL.md § Analogies)  
- [ ] Figures are static, labeled, and readable without interaction  
- [ ] Example data on flows  
- [ ] No ASCII main diagrams  
- [ ] Quiz quality rules  
- [ ] `pre` whitespace correct; JS parses if any  
- [ ] Path: `~/explanations/YYYY-MM-DD-explanation-*.html`  
