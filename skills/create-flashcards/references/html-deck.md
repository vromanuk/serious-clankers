# HTML deck page (create-flashcards)

**Default deliverable:** one self-contained HTML file — visual deck, every card copyable with formatting preserved, important terms in **bold**.

Load with `guide.md` when building the page.

---

## Output path

```text
~/explanations/YYYY-MM-DD-flashcards-<short-slug>.html
```

Create `~/explanations/` if needed. Use **today’s local date**. Do not put the file in a code repo unless the user asks.

Optional: also paste a short markdown summary in chat (path + card count + spine). Do **not** make chat-only markdown the main product unless the user says chat-only.

---

## Page requirements

| Rule | Detail |
|------|--------|
| Self-contained | Inline CSS + JS only. No CDN, fonts, images, packages. |
| Always-visible Q/A | No flip, no click-to-reveal, no hide-answer (unless user asks for flip UI). |
| **Copy on every card** | Button copies full card as plain text with newlines and bullets preserved. |
| **Bold important text** | Key terms, type names, outcomes, constraints in `<strong>` on the page; same emphasis reflected in copied text as `**…**` or kept as readable emphasis. |
| Themed groups | Sections by theme (`## Smart pointers`, …) with card list under each. |
| Readable on phone | Responsive layout. |
| Selectable text | Normal `user-select: text` on Q and A. |

---

## Bold what matters

On **Q and A**, wrap in `<strong>` (and use `**…**` in the copy payload):

- Type / API names: `Box<T>`, `RefCell`, `Rc`  
- Critical outcomes: **panic at runtime**, **compile error**, **single owner**  
- Constraints: **single-threaded only**, **heap not stack**  
- Contrast poles: **compile time** vs **runtime**  
- The one-line spine of the answer (or its key phrase)

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

- Store a plain-text payload for copy in `data-copy` on the article **or** build it in JS from Q text + A text (prefer `data-copy` so bold becomes `**bold**` and bullets stay clean).  
- `card-a-body` uses `white-space: normal` with real `<p>`/`<ul>` for display; copy string uses newlines + `- ` bullets.

### Recommended `data-copy` format

```text
Q: Box vs Cell — when each?

A:
**Box<T>** is single ownership of heap data (or a fixed-size handle to unsized/recursive data).

**Cell<T>** is interior mutability for **Copy** values: mutate through `&T` when you need replace/get, not shared ownership.

- Reach for **Box** when the issue is where data lives / recursive size.
- Reach for **Cell** when mutating behind a shared reference for Copy payloads.
```

Quizlet and notes apps paste this cleanly. Preserve blank lines between blocks.

---

## Copy button JS (required pattern)

Every `.copy-btn` must:

1. Read that card’s full text (prefer `article.dataset.copy`, else assemble from DOM).  
2. Write with `navigator.clipboard.writeText(text)`.  
3. Fallback: temporary `<textarea>`, `select()`, `document.execCommand('copy')`.  
4. Show brief feedback on the button (“Copied”) then restore label.  
5. Work offline; no network.

Escape HTML in page content. Do not put unescaped `</script>` in strings.

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
header     title, one-line spine, card count, source note
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
- [ ] Every card has a **Copy** control that preserves newlines and bullets  
- [ ] Important terms use `<strong>` on page and `**…**` in copy payload  
- [ ] Q and A both always visible  
- [ ] No external assets  
- [ ] Mobile-readable  
- [ ] JS has no parse errors; copy works in a local file open  

---

## Chat handoff

- Absolute path to the HTML file.  
- Card count + theme list.  
- One-sentence spine of the deck.  
- Do not dump every card into chat (user opens the page / copies from there).
