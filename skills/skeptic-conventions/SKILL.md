---
name: skeptic-conventions
description: >
  Stage 6 of skeptic: plain words in code, one function per task, clear Rust
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
   - Report `composition: ok` or composition findings  
4. **Rust style** — ownership, types at edges, errors, async at the edge, needless clones; don’t re-argue `clippy`/`rustfmt`; allow a measured hot-path tradeoff with a short local why  
5. **Linters** — run the tools; don’t re-litigate their output here  

## Do not

- Comment quality as the main job (→ comments)  
- Naming as the main job (→ naming)  
- Absolute ban list (→ hard-rules)  
- Whole system redesign (→ purpose / architecture)  
- Pure vs IO placement (→ testability)  

## Output section title

`## 6. Conventions`
