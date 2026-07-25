---
name: skeptic-conventions
description: >
  Stage 4 of skeptic: comments, plain language, composition (one function per
  task), Rust form. Use when skeptic runs or the user asks for
  conventions/comments/composition/Rust-form review only.
---

# Skeptic conventions

**Question:** Does the code follow written form rules: plain language, local reasoning, one function per task, Rust craft?

## Load (on demand)

1. `references/write-code.md` — comments + § Composition  
2. `references/comments.md`, `references/design-headers.md` — comment samples  
3. `references/rust-craft.md` — plain Rust form samples  

## Do

1. **Comments**  
   - Prefer names/types that carry intent; comments for what those still hide  
   - Documentation comments (`//!` / `///`): design and intent of the surface  
   - Implementation comments (`//`): non-obvious choices/hazards — not narrating the next line  
   - Design headers: purpose + given/expected, **no type line**  
   - Flag restating next lines; stale/lying comments  
2. **Language** — plain words; no coined pattern nicknames in names/comments  
3. **Reader over author** — flag clever-but-opaque forms when a plainer form would read better (unless measured perf with in-place evidence)  
4. **Composition (one function per task)** — required when multi-step bodies are in scope:  
   - One task or several packed into one body?  
   - Thin composer + named steps when it helps?  
   - Real step names (not `helper1`)?  
   - One-use rename of a loop → flag inline  
   - Report `composition: ok` or composition findings  
5. **Rust form** — ownership, types at edges, `Result`/errors, async at the edge, needless clones; not pedantic clippy tourism; allow measured hot-path tradeoffs with in-place evidence  
6. **Linters** — do not re-litigate `rustfmt` / `clippy`; run tools instead  

## Do not

- Absolute ban list (→ hard-rules)  
- Full system redesign (→ purpose / architecture)  
- Pure vs IO as API stamps on docs (→ testability judges that)  

## Output section title

`## 4. Conventions`
