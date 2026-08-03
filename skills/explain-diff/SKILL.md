---
name: explain-diff
spec_hash: 55ed75f1cacf
description: >
  Explain a code change, branch, or PR from first principles: background,
  intuition, alternatives, then code walkthrough. Use when the user wants to
  understand a diff deeply (not a PR body alone, not full skeptic review).
  Markdown by default; optional self-contained HTML. No quiz.
---

# Explain diff

Teach the reader what a **real code change** does and why. Mental model first; details after. Ground everything in the actual diff and surrounding code.

Not a PR blurb (`pr-description`). Not multi-lens review (`skeptic`). Not a free-floating concept class (`explain-topic`) unless the change is only the hook.

## Workflow

1. **Inspect the change**  
   - Scope: branch vs base, PR, or paths the user named.  
   - Committed work: merge-base, `git log`, `git diff --name-status`, enough of the patch to be accurate.  
   - Read surrounding code for background (callers, component roots, related types).  
   - Do not invent files, behaviors, or tests.  
2. **Name the real need** in plain words (label *assumed* if only inferred).  
3. **Draft** in the section order below (load `references/shape.md` for depth and HTML rules).  
4. **Deliver** markdown by default; HTML only if asked (or they clearly want a rich page).  

## Section order (always)

### 1. Background

We do not know how much the reader knows.

1. **Deep background** (beginners) — existing system enough to stand up from zero. Mark skippable if they already know it.  
2. **Narrow background** — only what this change touches.  

Explore surrounding code before narrowing.

### 2. Intuition (before detail)

**First principles and essence** — not a file list.

- Real need → core idea of the change.  
- Analogy if it helps; say where it breaks.  
- Toy data: small concrete before/after.  
- Diagrams liberally (ASCII and/or simple HTML). Prefer a few reusable diagram families (e.g. one flow picture, one before/after).  
- High-level idea first; mechanism detail later.  

### 3. Alternatives

When the design is non-obvious (or tradeoffs matter):

- 2–3 **real** options (including what landed).  
- Pros / cons / when each is worse.  
- Why this change took its path (or *assumed*).  

Skip a long alternatives essay for tiny one-line fixes; one sentence is enough.

### 4. Code

High-level walkthrough **after** the model is clear.

- Group/order by idea or data flow, not only path sort.  
- Key snippets only when they carry the point.  
- Note contracts, edges, and what did not change.  

### Optional close

One line: “You should be able to …” — understanding check, **not** a quiz.

## Output

| Default | Optional |
|---------|----------|
| Markdown (chat or file) | Self-contained HTML page |

**HTML** (when asked): one file, CSS (and JS only if useful — no quiz), long page + TOC, responsive enough for a phone. Path **outside the repo**, name starts with **today’s date** `YYYY-MM-DD-` (e.g. `/tmp/2026-07-27-explanation-branch-slug.html`). Code in `<pre>` or any block with `white-space: pre` / `pre-wrap`. Callouts for key ideas and edges. Details: `references/shape.md`.

## Voice

- Clear, calm technical prose (classic style, smooth transitions).  
- Plain words. Direct.  
- Facts vs assumptions.  

## Never

- Interactive quiz, flashcards, or multi-choice tests.  
- Open with a raw file dump.  
- Invented features or test runs.  
- Treat this as “implement the feature.”  
- Corporate/HR padding.  
- Mermaid (plain text / HTML structure only).  

## Related

| Job | Skill |
|-----|--------|
| Short PR body (Why = need + intuition; How = mechanism) | `pr-description` |
| Multi-lens review | `skeptic` |
| Concept only | `explain-topic` |
