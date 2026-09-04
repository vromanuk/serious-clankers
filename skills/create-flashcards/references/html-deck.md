# HTML deck page (create-flashcards)

**Default deliverable:** one self-contained HTML file — visual deck, every card copyable with formatting preserved, important terms in **bold**.

Load with `guide.md` when building the page.

---

## Output path

```text
~/explanations/YYYY-MM-DD-flashcards-<short-slug>.html
```

Create `~/explanations/` if needed. Use **today’s local date**. Do not put the file in a code repo unless the user asks.

Optional: also paste a short markdown summary in chat (path + card count + one-line summary). Do **not** make chat-only markdown the main product unless the user says chat-only.

---

## Page requirements

| Rule | Detail |
|------|--------|
| Self-contained | Inline CSS + JS only. No CDN, fonts, images, packages. |
| Always-visible Q/A | No flip, no click-to-reveal, no hide-answer (unless user asks for flip UI). |
| **Copy on every card** | Button copies the full card as **formatted rich text — real bold/code, not markdown** (see "Copy = formatted text" below). Newlines and bullets preserved. |
| **Bold important text** | Key terms, type names, outcomes, constraints in `<strong>` on the page. Copied output carries the emphasis as **real bold** in the rich (`text/html`) flavor; the plain-text fallback has the `**`/backtick markers **stripped** (clean text, no markdown). |
| Themed groups | Sections by theme (`## Smart pointers`, …) with card list under each. |
| Readable on phone | Responsive layout. |
| Selectable text | Normal `user-select: text` on Q and A. |

---

## Bold what matters

On **Q and A**, wrap in `<strong>` on the page. Keep `**…**` markers in the **card source** only — they are used to build the two copy flavors (real bold in `text/html`, stripped in `text/plain`), never pasted as-is:

- Type / API names: `Box<T>`, `RefCell`, `Rc`  
- Critical outcomes: **panic at runtime**, **compile error**, **single owner**  
- Constraints: **single-threaded only**, **heap not stack**  
- Contrast poles: **compile time** vs **runtime**  
- The one-line main point of the answer (or its key phrase)

Do **not** bold entire paragraphs. Prefer a few strong anchors per card so the eye finds the model.

---

## Card DOM shape (required)

```html
<article class="card" data-card-id="1">
  <header class="card-head">
    <span class="card-tag">contrast</span>
    <button type="button" class="copy-btn" aria-label="Copy card">Copy</button>
  </header>
  <p class="card-q">
    <span class="qa-label">Q:</span>
    <span class="card-q-text"><strong>Box</strong> vs <strong>Cell</strong> — when each?</span>
  </p>
  <div class="card-a">
    <span class="qa-label">A:</span>
    <div class="card-a-body">
      <p><strong>Box&lt;T&gt;</strong> is single ownership of heap data (or a fixed-size handle to unsized/recursive data).</p>
      <p><strong>Cell&lt;T&gt;</strong> is interior mutability for <strong>Copy</strong> values: mutate through <code>&amp;T</code> when you need replace/get, not shared ownership.</p>
      <ul>
        <li>Reach for <strong>Box</strong> when the issue is where data lives / recursive size.</li>
        <li>Reach for <strong>Cell</strong> when mutating behind a shared reference for Copy payloads.</li>
      </ul>
    </div>
  </div>
</article>
```

- Keep the card source (Q text + A text, with `**bold**` / `` `code` `` markers) in JS data, and build **both** copy flavors from it at click time.  
- `card-a-body` uses `white-space: normal` with real `<p>`/`<ul>` for display.

**Recommended: one data array as source of truth.** Put all cards in a single JS array (`const CARDS = [{theme, tag, q, a}, …]`) and **render the visible cards, the search index, and both copy flavors from it**. This avoids duplicating Q/A between display and copy, and makes incremental adds a one-line change. `a` uses `\n` for line breaks and `- ` for bullets; a small renderer turns it into `<p>`/`<ul>` for display and into the two copy flavors.

**Validate before shipping/opening.** After writing or editing the array, confirm the embedded script parses (e.g. extract the `<script>` body and evaluate the array with a `node -e` one-liner, or open the file) so a stray quote or backtick never silently breaks the whole page. Note: keep backticks and `${` out of card text if the array uses template literals.

---

## Copy = formatted text, not markdown (required)

The copy button must put the card on the clipboard so it **pastes as formatted rich text** (real bold and code) in rich editors (Docs, Notion, Slack, email), and as **clean plain text with no markdown markers** everywhere else. Never paste literal `**` or backticks.

Write **two clipboard flavors** in one copy:

- `text/html` — render `**bold**` → `<strong>`, `` `code` `` → `<code>`, newlines → `<br>`, `- ` → bullets. This is what gives real formatting on paste.  
- `text/plain` — **strip** the `**`/backtick markers; keep newlines, blank lines, and `- ` bullets. This is the fallback for plain fields.

### Required pattern

```html
<script>
function esc(s){ return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
function toHtml(s){ // escaped text -> real <strong>/<code>, <br>, bullets
  return esc(s).replace(/\*\*([^*]+)\*\*/g,'<strong>$1</strong>')
               .replace(/`([^`]+)`/g,'<code>$1</code>')
               .split('\n').map(l => l.replace(/^- /,'&bull;&nbsp;')).join('<br>');
}
function stripMd(s){ return s.replace(/\*\*([^*]+)\*\*/g,'$1').replace(/`([^`]+)`/g,'$1'); }

function cardHtml(q,a){
  return '<div style="font-family:system-ui,Segoe UI,Roboto,Arial,sans-serif;font-size:14px;line-height:1.5">'
       + '<p style="margin:0 0 8px"><strong>Q:</strong> ' + toHtml(q) + '</p>'
       + '<p style="margin:0"><strong>A:</strong><br>' + toHtml(a) + '</p></div>';
}
function cardPlain(q,a){ return 'Q: ' + stripMd(q) + '\n\nA:\n' + stripMd(a); }

function copyRich(html, plain, btn){
  const done = () => { btn.textContent = 'Copied'; setTimeout(()=>btn.textContent='Copy',1200); };
  if (navigator.clipboard && window.ClipboardItem) {
    try {
      const item = new ClipboardItem({
        'text/html':  new Blob([html],  {type:'text/html'}),
        'text/plain': new Blob([plain], {type:'text/plain'})
      });
      navigator.clipboard.write([item]).then(done, () => execFallback(html, plain, done));
      return;
    } catch(e) {}
  }
  execFallback(html, plain, done);
}
// Fallback keeps formatting: select a hidden contenteditable, then execCommand('copy').
function execFallback(html, plain, done){
  const d = document.createElement('div');
  d.contentEditable = 'true';
  d.style.position='fixed'; d.style.left='-9999px'; d.style.whiteSpace='pre-wrap';
  d.innerHTML = html; document.body.appendChild(d);
  const r = document.createRange(); r.selectNodeContents(d);
  const sel = window.getSelection(); sel.removeAllRanges(); sel.addRange(r);
  let ok=false; try { ok = document.execCommand('copy'); } catch(e){}
  sel.removeAllRanges(); document.body.removeChild(d);
  if (ok) return done();
  const ta = document.createElement('textarea'); ta.value = plain;    // last resort: clean plain text
  ta.style.position='fixed'; ta.style.opacity='0';
  document.body.appendChild(ta); ta.select();
  try { document.execCommand('copy'); } catch(e){}
  document.body.removeChild(ta); done();
}
</script>
```

Every `.copy-btn` calls `copyRich(cardHtml(q,a), cardPlain(q,a), btn)`, shows brief “Copied” feedback, and works offline. A "Copy all" button joins the visible cards (rich flavor with `<hr>` between; plain flavor with a text divider). Escape page content; do not put unescaped `</script>` in strings.

### Rendering rules (or the copy mangles code / loses spacing)

- **Protect `` `code` `` first**, then `**bold**`, then `*italic*`. Otherwise a `*` inside code (e.g. `` `*mut T` ``, `` `*const` ``, `` `*b` ``) gets eaten as italics and leaves stray `*` in the paste. Extract code spans to placeholders, transform, then restore. Applies to **both** the page renderer and the plain-text stripper.  
- **Bold may wrap italics**: match bold non-greedy (`/\*\*([\s\S]+?)\*\*/`) before italics; match italics as `/\*([^*\n]+?)\*/`.  
- **Rich (`text/html`) flavor = real blocks**: build `<p>` paragraphs and a real `<ul><li>` list from the answer — **not** a run of `<br>`. `<br>` runs collapse into one line when pasted into some editors.  
- **Code blocks in the copy flavor must carry inline styles + explicit breaks** or the paste loses indentation and line breaks (Google Docs is the worst offender). The clipboard `text/html` is a standalone fragment with **no page CSS**, so a bare `<pre><code>` renders as one collapsed line. In the copy flavor render each code block as an **inline-styled** block (monospace font, background, padding) with **newlines → `<br>`** and **leading/every space → `&nbsp;`** so shape survives everywhere. Likewise give inline `` `code` `` and `**bold**` inline styles in the copy flavor. Keep the display renderer on CSS classes; only the copy path needs inline styles. Pattern:

```js
function codeCopyHtml(codeText){
  var lines = codeText.split('\n').map(l => esc(l).replace(/ /g,'&nbsp;'));
  return '<pre style="font-family:ui-monospace,Menlo,Consolas,monospace;font-size:13px;'
       + 'line-height:1.5;background:#2b2a27;color:#f3efe6;padding:10px 12px;border-radius:8px;'
       + 'white-space:pre-wrap;word-break:break-word;margin:8px 0">' + lines.join('<br>') + '</pre>';
}
// blocksHtml(a, copy): when copy, emit codeCopyHtml(...) and inline-styled <strong>/<code>; else use CSS classes.
```

  The `text/plain` fallback keeps real spaces for code indentation (do not turn them into `&nbsp;` there).  
- **Plain flavor = readable spacing**: put a **blank line between adjacent bullets** and keep blank lines between blocks, so notes/Quizlet paste stays structured.  
- **Tags are for the search filter only** (`data-search`). Do **not** render tag chips on the card — they repeat words already in the question/section and read as redundant.

---

## Suggested layout / CSS tokens

Reuse a soft paper theme (similar to explain-topic kit):

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
  --shadow: 0 8px 24px rgba(28, 27, 25, 0.08);
  --radius: 14px;
  --sans: system-ui, "Segoe UI", sans-serif;
  --mono: ui-monospace, Menlo, Consolas, monospace;
}
```

Ideas:

- Page: max-width ~48–52rem, padding, theme sections with `h2`.  
- Card: paper, border, left accent bar, shadow; spacing between cards.  
- Q: slightly larger / bolder than body.  
- A: soft background (`--accent-soft` or `--good-soft`) so answer region is obvious but always readable.  
- `code` / mono for type names when not bolded.  
- `.copy-btn`: clear click target; visible focus ring.  
- Optional filter chips by tag (show/hide whole cards only).

---

## Page skeleton

```text
header     title, one-line summary, card count, source note
nav        jump links to themes (optional)
main
  section#theme-…
    h2 theme
    article.card × N
footer     path hint / generated date
script     copy handlers
```

Optional top **table of themes** with counts.

---

## Quality checklist (HTML)

- [ ] File under `~/explanations/YYYY-MM-DD-flashcards-*.html`  
- [ ] Every card has a **Copy** control; it pastes as **formatted rich text (real bold/code), not markdown** — verify by pasting into a rich editor  
- [ ] Copy writes both `text/html` (real formatting) and a clean `text/plain` fallback (no `**`/backticks); newlines + bullets preserved in both  
- [ ] Important terms use `<strong>` on page (source keeps `**…**` markers only to build the two copy flavors)  
- [ ] Copy protects `` `code` `` before bold/italics (no stray `*` from `*mut`/`*const`/`*b`); rich flavor uses real `<p>`/`<ul>`, plain flavor has blank lines between bullets  
- [ ] **Code blocks paste with formatting**: copy flavor styles code inline with `<br>` line breaks + `&nbsp;` indentation (bare `<pre><code>` collapses) — verify by pasting a code card into a rich editor  
- [ ] No visible tag chips (tags only in `data-search`); no coined nicknames/metaphors; no redundant “(plain)”-style qualifiers in questions  
- [ ] Q and A both always visible  
- [ ] No external assets  
- [ ] Mobile-readable  
- [ ] JS has no parse errors; copy works in a local file open  

---

## Chat handoff

- Absolute path to the HTML file.  
- Card count + theme list.  
- One-sentence summary of the deck.  
- Do not dump every card into chat (user opens the page / copies from there).
