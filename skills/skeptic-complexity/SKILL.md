---
name: skeptic-complexity
description: >
  Stage 3 of skeptic: does the change make code harder to understand or change?
  Deep vs shallow units, information leakage, pass-throughs, overexposure,
  special-general mixture, error and special-case design, split vs join,
  nonobvious code, and change shape (Ousterhout's red flags). Use when skeptic
  runs or the user asks for a complexity / design-depth review only.
---

# Skeptic complexity

**Question:** Does this change make the code harder to understand or change — and is there a simpler design that removes that cost?

## Load

1. `references/complexity.md` — always when the diff changes code: model, red flags with examples and limits, error techniques, review checklist  

## Do

1. Walk `complexity.md` § 14 Review checklist on each **new or changed** unit (function, type, module, error path). Look at callers too: a cost often shows at the call site.  
2. **Depth:** flag shallow functions and types, pass-through methods, pass-through variables, and wrappers that only forward. Not a finding: a dispatcher, several implementations of one trait, a long function with a simple signature that reads top to bottom.  
3. **Leakage:** flag a decision (format, rule, ordering, storage shape) known in two places — including places no interface links — and modules split by when they run while sharing knowledge.  
4. **Common case:** flag arguments almost every caller sets the same way, values the module could derive, and new configuration the module could compute. Never ask to hide what callers need for correctness.  
5. **General vs special:** flag a general mechanism that contains one caller's special case, and special-case checks where an empty or default value would let the normal path handle it.  
6. **Errors:** for each new error, ask in order: can the operation be redefined so it disappears; handled lower down; handled with many others in one place; or should it abort? Flag errors no caller can act on. Also flag the opposite: an error swallowed that callers need.  
7. **Split / join:** flag conjoined pieces (each readable only with the other), splits made for length alone, and repeated code. Length alone is not a finding.  
8. **Design signals:** a name that is hard to pick, or a complete doc comment that must be long, means a design problem — report the design side here; leave the wording to naming / comments.  
9. **Obvious:** flag tuples or generic containers for values with meaning, hidden side effects that break expectations, and callbacks with no note on when they run.  
10. **Change shape:** flag a special case, flag, or dependency bolted on where fitting it into the design was cheap. For a new public interface or major structure with no visible alternative, suggest one concrete, really different design.  
11. **Findings:** red flag name (or principle) · symptom (change amplification / cognitive load / unknown unknowns) · path:line · smallest change that removes it. LETTER options when the fix is a real design fork.  
12. When code is in scope: `complexity: ok` (one line: the main units checked) or findings. Pure renames or formatting with no structural change → `none` with one-line why.  

## Do not

- Component layout, data ownership, or the job struct's public face (→ architecture); report here only depth and leakage inside or across units  
- Comment wording (→ comments) or name wording (→ naming)  
- Pure vs IO placement (→ testability) or test craft (→ unit-tests)  
- Ask for cleanup of code the change did not touch; offer it as a follow-up  
- Flag a red flag where the reference lists it as a limit (e.g. hiding information callers need, splitting by phase when phases share nothing)  
- Taste findings with no symptom  
- Speed changes without measurements (hot-path API costs → architecture)  

## Output section title

`## 3. Complexity`
