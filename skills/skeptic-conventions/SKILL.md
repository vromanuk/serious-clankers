---
name: skeptic-conventions
description: >
  Stage 9 of skeptic: plain words in code, one function per task, clear Rust
  style. Use when skeptic runs or the user asks for composition or Rust style
  review only. Not for comments, naming, or hard-rules.
---

# Skeptic conventions

**Question:** Plain words? One function per task? Clear Rust style?

Comments → stage **comments**. Naming → stage **naming**.

## Load (on demand)

1. `references/write-code.md` — composition, write for the reader  
2. `references/rust-craft.md` — control flow and clear style samples  

## Do

1. **Language** — plain words; no made-up pattern names in code or remaining text  
2. **Easy to read** — flag clever but hard-to-follow code when a simpler shape would be clearer (unless a real measured hot path has a short why-comment)  
3. **One function per task** when multi-step bodies are in scope:  
   - Several jobs packed into one body?  
   - Thin outer function + named steps when that helps?  
   - Real step names (not `helper1`)?  
   - Helper that only renames a loop → say inline it  
   - Split only when each piece reads on its own; length alone is not a reason (conjoined pieces → `skeptic-complexity`)  
   - Report `composition: ok` or composition findings  
   - Interior split lives here; **exported** leaves (`opening` / `body` as `pub`, `load_*` / `validate_*` / `write_*` on the job root) → architecture `### Component API`  
4. **Rust style** — ownership, types at edges, errors, async at the edge, needless clones; don’t re-argue `clippy`/`rustfmt`; allow a measured hot-path tradeoff with a short local why  
5. **Assertions** — flag an invariant the code depends on with no assertion and no type enforcing it, compound assertions that hide which condition failed, and an expensive `assert!` on a hot path where `debug_assert!` fits. Panics on outside input are error design (→ complexity). Detail: `references/rust-craft.md` § Assertions  
6. **Settings written out** — flag a client or consumer built on library defaults for timeouts, commit behavior, offset reset, isolation, retries, or delivery guarantees the code relies on (`rust-craft.md` § Pass settings explicitly)  
7. **One copy of each fact** — flag a stored field that duplicates a value derivable from others (a count beside its collection) without a measured reason, and a check whose result is used after code that can change what was checked (`rust-craft.md` § One copy of each fact)  
8. **Linters** — run the tools; don’t re-litigate their output here  

## Do not

- Comment quality as the main job (→ comments)  
- Naming as the main job (→ naming)  
- Absolute ban list (→ hard-rules)  
- Whole system redesign, or whether composition leaves are public (→ architecture `### Component API`)  
- Pure vs IO placement (→ testability)  

## Output section title

`## 9. Conventions`
