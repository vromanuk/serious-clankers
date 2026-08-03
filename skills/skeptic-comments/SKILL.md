---
name: skeptic-comments
description: >
  Stage 5 of skeptic: are comments clear, needed, and about why (benefit /
  design reason) — not restating the code or bare rules without explanation.
  Use when skeptic runs or the user asks for comment review only. Not for
  naming, composition, or hard-rules.
---

# Skeptic comments

**Question:** Are comments clear English, actually needed, and about **why** (reason, benefit, why-not) — not a restate of what the next line does, and not a design rule with no explanation?

## Load (on demand)

1. `references/comments.md` — **why bar**, module `//!` template, bad/good examples  
2. `references/google-comments.md` — Google comment ideas (any language) + review list  
3. `references/design-headers.md` — purpose (+ why) + given/expected, no type line  

## Do

1. Prefer clear names and types; comment only what those still hide.  
2. **When reviewing comments:**  
   - Clear English?  
   - Needed at all?  
   - Explains **why** / **benefit** / **why not the obvious alternative** — not only **what**, and not a bare rule with no reason?  
   - Hard to follow code → simplify the code first (OK to explain *what* for regex or hard algorithms).  
   - Keep only what the code cannot say (reasoning, scars, rules **with** their reason).  
3. **Public docs vs body comments:**  
   - On the type/function (`///`): how to **use** it; **why this shape** when non-obvious.  
   - On the **module** (`//!`): follow `comments.md` template — **role → intuition → why → rules → entrypoints → related → flow** (skip empty parts).  
   - Inside the body (`//`): tricky **how/why** — do not copy the public blurb into the body.  
4. Flag module docs that only list policies (“do X, not Y”) with no benefit or rejected alternative when that is a real design choice.  
5. On types and functions: purpose and contracts when not obvious. On fields: special values the type doesn’t show.  
6. **Do not state the obvious** — no comment that only repeats the next line.  
7. TODOs: who owns it and enough context; if time-based, a clear end condition.  
8. Design headers: purpose (+ why if a choice) + given/expected, **no type line**.  
9. Flag: restating lines, unexplained rules, useless comments, wrong/stale comments, hard-to-read comment text.  

## Do not

- Naming review as the main job (→ naming)  
- Composition / how functions are split (→ conventions)  
- Absolute ban list (→ hard-rules)  
- Who owns which component (→ architecture)  
- Demand a novel on every private module  

## Output section title

`## 5. Comments`
