---
name: skeptic-purpose
description: >
  Stage 1 of skeptic: real need, structure diagrams, whether the change makes
  sense, serious bugs, alternatives with pros/cons. Use when skeptic runs or
  the user asks only for purpose/design-sense review.
---

# Skeptic purpose

**Question:** Why does this change exist, and does the approach match a real need?

## Do

1. **Real need** — one plain sentence. Label assumption if unverified.  
2. **Structure diagrams** (plain text / ASCII or component diagrams — no Mermaid):  
   - **L0** neighbors / system  
   - **L1** components in scope  
   - **L2** change / data / control flow  
   - Optional **L3** one hot path  
3. **Does this shape make sense?** 2–5 sentences: strong / awkward / over-under. Fact vs judgment.  
4. **Serious correctness bugs** in the approach (wrong quantity, broken contract, silent empty results, …).  
5. **Alternatives** for material design forks: 2–3 real options, recommended first, pros/cons/gain/worse-when.  
6. If recommending implementation work: prefer chunked order (separate landings or commits), not one opaque mega-commit.  

## Do not

- Pure-core purity deep-dive (→ testability)  
- Component public/private catalog as the whole review (→ architecture)  
- Comment form nits (→ conventions)  
- Absolute ban checklist (→ hard-rules)  

## Output section title

`## 1. Purpose`
