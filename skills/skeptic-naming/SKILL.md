---
name: skeptic-naming
description: >
  Stage 6 of skeptic: are names clear — long enough to say what something is or
  does, without being hard to read. Use when skeptic runs or the user asks for
  naming review only. Not for comments, composition, or hard-rules.
---

# Skeptic naming

**Question:** Did they pick good names? Long enough to say what the thing is or does, not so long that it’s hard to read.

## Load (on demand)

1. `references/naming.md` — naming ideas (from Google, any language) + review list  

## Do

1. For each new or changed name that matters: does it say **what it is or does**?  
2. Flag names that are **too short or unclear** for how widely they’re used (odd abbreviations, dropped letters, vague `data` / `tmp` / `helper` on something others call).  
3. Flag names that are **too long** for how little they matter (essay names for loop locals; repeating what’s already in the type).  
4. **Wider use → clearer name.** A public function needs a fuller name than a local in a 5-line loop (`i` / `n` can be fine there).  
5. Functions that **do** work: name the action. Types and values: name the thing.  
6. **Plain words** — no made-up pattern labels as names.  
7. **Match this project’s style** (e.g. Rust: `snake_case` functions, `CamelCase` types). Don’t force another language’s style.  
8. If a comment only exists to explain a bad name → say **rename**, not “add more comment.”  
9. When naming is in scope: `naming: ok` (one line) or findings with path:line and a better name when easy.  

## Do not

- Comment wording as the main job (→ comments)  
- How big functions should be split (→ conventions)  
- Invent a new naming style against the local code  
- Absolute ban list (→ hard-rules)  

## Output section title

`## 6. Naming`
