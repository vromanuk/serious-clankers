---
name: skeptic-architecture
description: >
  Stage 2 of skeptic: component ownership, public vs private surface, data
  ownership, dependency direction, type-driven contracts at boundaries. Use when
  skeptic runs or the user asks for architecture/component review only.
---

# Skeptic architecture

**Question:** Who owns which job, how may pieces depend, and do types force the real contract at boundaries?

## Load (on demand)

- `../skeptic/references/personal-guidelines.md` § Architecture  
- `../skeptic/references/write-code.md` — names/types at the use site  
- `../skeptic/references/examples/type-driven.md` — worked type-boundary samples  
- `../skeptic/references/examples/rust-apis.md` — misuse-resistant APIs  

## Do

1. Name **components / modules** by job (not by technical layer).  
2. For each changed boundary: **public surface** vs **internal**; flag imports of someone else’s private surface.  
3. **Data ownership** — who writes which facts; no shared write free-for-all.  
4. **Compose at the edge** — app/binary wires components; no tangled internal graph.  
5. **Tool vs product:** small scripts stay local; no package theater.  
6. **Type-driven contracts** when new/changed APIs or shared results are in scope (below).  
7. Findings with file:line; options when a boundary or type should move.  

## Type-driven contracts

**Question:** Can a caller forget a real decision, or pass an impossible value, without the type system complaining?

| Prefer | Over |
|--------|------|
| Enum / newtype for mutually exclusive modes | Parallel `Option`s the caller must keep consistent |
| Non-empty / checked success type when empty is a product error | `Vec` / bare list + second `if empty` at each call site |
| Decision enum from pure code | Bool soup / magic strings the shell re-interprets |
| One constructor path with shared failure | Copy-pasted bail strings that can diverge |

**Ask on each changed boundary:**

1. What must the caller **not** forget?  
2. Is that in the **return/arg type**, or only in comments / a second helper?  
3. Do shared policies share **one type/constructor**?  
4. Is may-empty inventory a **different type/API** from must-have product paths?  

**Flag:** silent empty success; optional fields that only make sense together; duplicate empty-checks with the same string.

When APIs/results changed, include `type-driven: ok` (what the type forces) or type-driven findings.

## Do not

- Pure vs IO placement deep-dive (→ testability) unless it is boundary ownership  
- Comment wording (→ conventions)  
- Hard bans list (→ hard-rules)  
- Newtypes for every list when empty is a valid inventory outcome  

## Output section title

`## 2. Architecture`
