---
name: skeptic-hard-rules
description: >
  Stage 5 of skeptic: absolute bans and pass/fail checks only. Use when skeptic
  runs or the user asks for hard-rules / must-not review. Not for soft style
  taste or product design.
---

# Skeptic hard rules

**Question:** Does this diff violate a non-negotiable ban?

## Load

- **Only** `../skeptic/references/hard-rules.md` as the checklist (Skillet: shared under coordinator skill)  
- Optional: run deterministic tools (`clippy`, `rustfmt`, tests) and report pass/fail  

## Do

1. Walk `hard-rules.md` item by item against the diff / scoped files.  
2. Each hit: **blocker** severity (or must), file:line, exact rule id, smallest fix.  
3. If nothing hits: `none` for this section.  
4. Optional: “could this become a lint/CI check?” — plan only, no implement unless asked.  

## Do not

- Soft preferences (“might be clearer”)  
- Expanding into architecture or pure-core essays  
- Inventing new hard rules mid-review without adding them to `hard-rules.md` later  

## Output section title

`## 5. Hard rules`

Format:

```text
[blocker][evidence:policy,direct hard-rule:<id>] path:line — violation. fix: <smallest change>.
```
