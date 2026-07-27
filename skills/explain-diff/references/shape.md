# Explain-diff shape

Load after you have inspected the **real** branch/diff and enough surrounding code for background.

## Section order (always)

1. **Background** — deep (skippable) → narrow  
2. **Intuition** — first principles, essence, analogy, toy data, diagrams  
3. **Alternatives** — real options, pros/cons, why this path  
4. **Code** — high-level walkthrough, grouped by idea  

Optional short close: “you should be able to …” (one check, not a quiz).

No quiz. No flashcards.

---

## Background

### Deep (beginners — mark skippable)

Enough of the **existing system** to stand up from zero: what problem area this lives in, main pieces, data flow in plain words.

Explore surrounding code (callers, components, docs if present). Do not invent architecture.

### Narrow (this change)

Only paths, types, and contracts this diff actually touches.

---

## Intuition

Lead with **why this change exists** and the **core idea** (not every file).

- First principles: real need → smallest mechanism.  
- One strong analogy if it helps; say where it breaks.  
- Toy data: small concrete before/after.  
- Diagrams: few families, reused (e.g. one flow picture + one before/after).  
  - ASCII is fine.  
  - Simple HTML boxes/lists fine in markdown or full HTML page.  
  - Prefer clear structure over decoration.

---

## Alternatives

When the design is non-obvious (or the user cares about tradeoffs):

| Option | What | Pros | Cons | Worse when |
|--------|------|------|------|------------|
| A (recommended / what landed) | … | … | … | … |
| B | … | … | … | … |

No “do nothing” as a fake option. Label *assumed* if the need or rejected options are only inferred from the diff.

Tiny one-line fixes: one short “could have done X; chose Y because …” is enough.

---

## Code

After the model is clear:

- Group changes by **idea or flow** (not only path sort).  
- High level: what each group does and how it connects.  
- Show key snippets when they carry the point; not every hunk.  
- Call out contracts, edge cases, and what deliberately did **not** change.

---

## Output formats

### Default: markdown (chat or file)

Long page with headings. Diagrams as ASCII or small HTML fragments. Callouts as blockquotes or bold “Key idea” lines.

### Optional: self-contained HTML

When the user asks for a rich HTML page:

- One file: CSS + JS inline if needed (no quiz JS required).  
- One long page + table of contents; no top-level tabs.  
- Basic responsive layout.  
- **Path outside the code repo**, name starts with **today’s date** `YYYY-MM-DD-`:  
  - e.g. `/tmp/2026-07-27-explanation-my-branch.html`  
  - or `~/explanations/YYYY-MM-DD-….html` if you use a personal folder  
- Code: use `<pre>` (or any code container with `white-space: pre` / `pre-wrap`).  
- Before save: check code blocks keep newlines.  
- Diagrams: simple HTML, not ASCII-only requirement in HTML mode.  
- Callouts for key ideas and edge cases.

---

## Voice

- Clarity and calm flow (classic technical prose).  
- Plain words first. Smooth transitions between sections.  
- Facts vs assumptions.  

## Not this skill

| Job | Skill |
|-----|--------|
| Short PR body | `pr-description` |
| Multi-lens review | `skeptic` |
| Concept with no change | `explain-topic` |
