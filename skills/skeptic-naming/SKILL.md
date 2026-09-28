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
2. Flag names that are **too short to say which thing** — odd abbreviations, dropped letters, and ambiguous words even on a local.  
3. **Variables, fields, and parameters say which thing.** Flag `list`, `stale`, `read`, `data`, `items`, `result`, `value`, `tmp`. `read` is a verb, not a value. `stale` needs its noun (`stale_resume`). Detail: `naming.md` § Variable names.  
4. Flag names that are **too long** for how little they matter (essay names for loop locals; repeating what’s already in the type).  
5. **Wider use → clearer name.** A local is not an excuse for `list` or `stale`. What stays short: `i` / `n` in a tiny loop, and `ok` / `found` in a few lines when nothing else could match.  
6. **Function naming (stronger)** — apply `naming.md` § Function naming:  
   - **Every function starts with a verb** or a verb phrase. A bare noun is a value, not work (`next_fetch_offsets()` as a function, `resume_offsets()`, `file_list()`). Constructors (`new`, `from_*`) and names a trait requires stay exempt.  
   - Not an amoeba verb alone (`process` / `handle` / `do`).  
   - Bool returns (and bool fields that stand alone): **affirmative predicate** with `is_` / `has_` / `can_` (or `should_` / `are_` / `needs_` when clearer).  
7. **Do not turn a verb into a noun to name a value.** `resume` is a verb. `resume_offsets` does not say which offsets. When one type carries two values (the record's offset and the next offset to read), the function is the verb (`commit`, `get_next_offsets`) and the argument name says which value (`next_offsets`). Bare `offsets` is not enough. `fetch` is not required once `next` already means that.  
8. Types and values: name the **thing** (nouns).  
9. **Plain words** — no made-up pattern labels as names.  
10. **Match this project’s style** (e.g. Rust: `snake_case` functions, `CamelCase` types). Don’t force another language’s style.  
11. If a comment only exists to explain a bad name → say **rename**, not “add more comment.”  
12. **Public / API surface (stronger):** apply `naming.md` § Public API naming — soft nits stay here; absolute API interface bans are hard-rules (`HR-api-*`).  
13. When naming is in scope: `naming: ok` (one line) or findings with path:line and a better name when easy.  
14. If a hit is clearly an `HR-api-*` ban, note it and leave the formal id to hard-rules (or report once if running this stage alone).  

## Do not

- Comment wording as the main job (→ comments)  
- How big functions should be split (→ conventions)  
- Invent a new naming style against the local code  
- Full hard-rule walk (→ hard-rules); still flag soft API naming nits  

## Output section title

`## 7. Naming`
