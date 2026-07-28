---
name: skeptic-comments
description: >
  Stage 5 of skeptic: are comments clear, needed, and about why — not restating
  the code. Use when skeptic runs or the user asks for comment review only. Not
  for naming, composition, or hard-rules.
---

# Skeptic comments

**Question:** Are comments clear English, actually needed, and about **why** — not a restate of what the next line does?

## Load (on demand)

1. `references/google-comments.md` — Google comment ideas (any language) + review list  
2. `references/comments.md` — examples of good hazard / ownership comments  
3. `references/design-headers.md` — purpose + given/expected, no type line  

## Do

1. Prefer clear names and types; comment only what those still hide.  
2. **When reviewing comments:**  
   - Clear English?  
   - Needed at all?  
   - Explains **why** (or a decision), not **what** the code does?  
   - Hard to follow code → simplify the code first (OK to explain *what* for regex or hard algorithms).  
   - Keep only what the code cannot say (reasoning, scars, rules).  
3. **Public docs vs body comments:**  
   - On the type/function (`///`): how to **use** it.  
   - On the **module** (`//!`): purpose, key rules, **entrypoints**, related modules, short flow when multi-step (`comments.md` § Module docs).  
   - Inside the body (`//`): tricky **how/why** — do not copy the public blurb into the body.  
4. On types and functions: purpose and contracts when not obvious. On fields: special values the type doesn’t show (e.g. `-1` means unknown).  
5. **Do not state the obvious** — no comment that only repeats the next line.  
6. TODOs: who owns it and enough context; if time-based, a clear end condition.  
7. Design headers: purpose + given/expected, **no type line**.  
8. Flag: restating lines, useless comments, wrong/stale comments, hard-to-read comment text.  

## Do not

- Naming review as the main job (→ naming)  
- Composition / how functions are split (→ conventions)  
- Absolute ban list (→ hard-rules)  
- Who owns which component (→ architecture)  

## Output section title

`## 5. Comments`
