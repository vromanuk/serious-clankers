---
name: skeptic-naming
description: >
  Stage 7 of skeptic: are names clear — long enough to say what something is or
  does, without being hard to read. Use when skeptic runs or the user asks for
  naming review only. Not for comments, composition, or hard-rules.
---

# Skeptic naming

**Question:** Did they pick good names? Long enough to say what the thing is or does, not so long that it’s hard to read.

## Load (on demand)

1. `references/naming.md` — general names (Google) + **public API naming** (The API Book) + review list  

## Do

1. For each new or changed name that matters: does it say **what it is or does**?  
2. Flag names that are **too short or unclear** for how widely they’re used (odd abbreviations, dropped letters, vague `data` / `tmp` / `helper` on something others call).  
3. Flag names that are **too long** for how little they matter (essay names for loop locals; repeating what’s already in the type).  
4. **Wider use → clearer name.** A public function needs a fuller name than a local in a 5-line loop (`i` / `n` can be fine there).  
5. **Function naming (stronger)** — apply `naming.md` § Function naming:  
   - Work functions: **verb / verb phrase** (not bare nouns; not amoeba `process`/`handle` alone).  
   - Bool returns (and bool fields that stand alone): **affirmative predicate** with `is_` / `has_` / `can_` (or `should_` / `are_` / `needs_` when clearer).  
6. Types and values: name the **thing** (nouns).  
7. **Plain words** — no made-up pattern labels as names.  
8. **Match this project’s style** (e.g. Rust: `snake_case` functions, `CamelCase` types). Don’t force another language’s style.  
9. If a comment only exists to explain a bad name → say **rename**, not “add more comment.”  
10. **Public / API surface (stronger):** apply `naming.md` § Public API naming — soft nits stay here; absolute API interface bans are hard-rules (`HR-api-*`).  
11. When naming is in scope: `naming: ok` (one line) or findings with path:line and a better name when easy.  
12. If a hit is clearly an `HR-api-*` ban, note it and leave the formal id to hard-rules (or report once if running this stage alone).  

## Do not

- Comment wording as the main job (→ comments)  
- How big functions should be split (→ conventions)  
- Invent a new naming style against the local code  
- Full hard-rule walk (→ hard-rules); still flag soft API naming nits  

## Output section title

`## 7. Naming`
