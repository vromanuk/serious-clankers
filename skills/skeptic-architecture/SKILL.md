---
name: skeptic-architecture
description: >
  Stage 2 of skeptic: component-based layout (job-shaped components, public vs
  private surface, data ownership, edge composition), dependency direction, and
  type-driven contracts at boundaries. Use when skeptic runs or the user asks
  for architecture/component review only.
---

# Skeptic architecture

**Question:** Who owns which job, how is the project laid out into components, how may pieces depend, and do types force the real contract at boundaries?

**Default (build and review):** job-shaped **components**, not horizontal layers. Same default as pack `context/AGENTS.md` § Project layout — this stage enforces it on diffs and guides structure when the change introduces layout.

## Load (on demand)

1. `references/components.md` — **default component layout** (public surface at the root; optional naming conventions; Hombergs / reflectoring)  
2. `references/type-driven.md` — type-boundary samples when APIs/results change  
3. `references/rust-apis.md` — misuse-resistant API craft  

## Do

### Component layout (primary — default architecture)

1. Name **components** by **job** (billing, check-engine, …) — not by technical layer alone (`controllers/`, `services/`, `repositories/` as the whole story). Prefer this when **adding** structure as well as when reviewing.  
2. For each component: a clear **public surface** (what outsiders may use) vs **private implementation**. In Rust, that is usually the root `mod.rs` / `lib.rs` defining/`pub use` of the public interface; child modules stay private unless re-exported.  
3. **Do not require** folders named `api/` or `internal/` — those are optional packaging labels. Flag missing *boundaries*, not missing folder names.  
4. Flag **imports past another component’s public surface** into private paths (same job as hard-rule `HR-private-import`).  
5. Nested helpers / sub-parts stay **inside** the component; they are not part of the public re-export unless intentional.  
6. **Data ownership** — who writes which facts/tables; no shared write free-for-all across jobs.  
7. **Compose at the edge** — app/binary/server wires components; no tangled mesh of private paths.  
8. **Tool vs product:** small scripts stay local; no package theater for a one-shot.  
9. When layout is in scope, report `components: ok` (one line: owners + surfaces) or layout findings with path:line.  

### Type-driven contracts (when APIs/results change)

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

### Findings

- Evidence at file:line (or import path).  
- LETTER options when a **boundary or ownership** should move (real alternatives).  

## Do not

- Pure vs IO placement deep-dive (→ testability) unless it is boundary ownership  
- Comment wording (→ comments)  
- Naming quality alone (→ naming)  
- Full hard-rule walk (→ hard-rules); still flag private-import as architecture  
- Newtypes for every list when empty is a valid inventory outcome  
- Demand multi-crate / multi-module packaging theater for a tiny single-site change  
- Demand literal `api/` / `internal/` folders when the root module already defines a clear public surface  

## Output section title

`## 2. Architecture`
