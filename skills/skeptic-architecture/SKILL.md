---
name: skeptic-architecture
description: >
  Stage 2 of skeptic: component-based layout (job-shaped components, public vs
  private surface, use-case orchestration surface not stray helpers, data
  ownership, edge composition), dependency direction, and type-driven contracts
  at boundaries. Use when skeptic runs or the user asks for architecture/component
  review only.
---

# Skeptic architecture

**Question:** Who owns which job, how is the project laid out into components, is the public surface a **job struct with use-case methods** (deep) or a bag of steps (shallow), how may pieces depend, and do types force the real contract at boundaries?

**Default (build and review):** job-shaped **components**, not horizontal layers. Same default as pack `context/AGENTS.md` § Project layout — this stage enforces it on diffs and guides structure when the change introduces layout.

**Layers ban ≠ no public orchestration:** app-wide `services/` folders stay out; **one public job struct + methods** stay in. Interior = pure-core/shell. Depth: `design-components` + `pure-core.md`.

## Load (on demand)

1. `references/components.md` — **default component layout** (public use-case face; Hombergs; review flags)  
2. `../design-components/references/guide.md` — when designing/growing a component interior (orchestration + sans-IO)  
3. `../design-components/references/philosophy-of-design.md` — when shallow vs deep face, leakage, or pass-through APIs are in scope  
4. `../skeptic-testability/references/pure-core.md` — when pure vs IO placement is in scope  
5. `references/type-driven.md` — type-boundary samples when APIs/results change, and when the diff adds a struct  
6. `references/rust-apis.md` — misuse-resistant API craft  

## Do

### Component layout (primary — default architecture)

1. Name **components** by **job** (billing, check-engine, …) — not by technical layer alone (`controllers/`, `services/`, `repositories/` as the whole story). Prefer this when **adding** structure as well as when reviewing.  
2. For each component: a clear **public surface** (what outsiders may use) vs **private implementation**. In Rust, that is usually the root `mod.rs` / `lib.rs` defining/`pub use` of the public interface; child modules stay private unless re-exported.  
3. **Public surface = job struct + methods** — flag many exported step-helpers (`load_*`, `validate_*`, `write_*`) or free orchestration `pub fn`s that re-pass the same deps; prefer methods on one public handle.  
4. Flag **handlers/CLI that own multi-step load→decide→save** when a component should own that orchestration.  
5. **Do not require** folders named `api/` or `internal/` — those are optional packaging labels. Flag missing *boundaries*, not missing folder names. **Do not flag** a private `service` / use-case module inside a job.  
6. Flag **imports past another component’s public surface** into private paths (same job as hard-rule `HR-private-import`).  
7. Nested helpers / sub-parts stay **inside** the component; “one function per task” does not mean every task is `pub`.  
8. **Data ownership** — who writes which facts/tables; no shared write free-for-all across jobs.  
9. **Compose at the edge** — app/binary/server wires components; no tangled mesh of private paths.  
10. **Tool vs product:** small scripts stay local; no package theater for a one-shot.  
11. When layout is in scope, report `components: ok` (one line: owners + surfaces) or layout findings with path:line.  
12. **Public data types live in `types`.** A `pub struct`, `pub enum`, or `pub type` that callers name is defined in that crate’s `types.rs` or `types/` module. A `pub use` from another file does not count. If `types` does not exist, the finding says add it and move the type there. The crate root may re-export it so the short path stays. Detail: `references/components.md` § Public data types.  

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

### New struct vs one that already exists

When the diff **adds a struct** (or a payload enum), look for an existing struct or enum variant in this crate — especially under `types` — with the same role and the same fields under different names.

**Flag it.** Do not treat a doc comment that distinguishes the new struct from some *other* type as enough. Do not pick the merge silently. The finding names both types and gives letter options:

- **Reuse** the existing type. Put the extra fact beside it, not in a second copy of every field.  
- **Generalize** the existing type so both stages use it (one record, one payload enum).  
- **Keep both** only when collapsing them would lie. Name the fact that must not exist on the earlier type. Renaming `offset` to `record_offset`, or `payload` to `body`, is not that fact.

Detail and the broker-record example: `references/type-driven.md` § New struct vs one that already exists.

**Not a flag:** a wrapper that holds the existing type plus one new fact; two structs that share a word but not the fields; a newtype whose only job is an invariant the other type does not have.

When APIs/results changed, or a struct was added, include `type-driven: ok` (what the type forces, and that no existing type was copied) or type-driven findings.

### High-level API design (when a component's API is designed or changed)

Part of component-based architecture: the public API is the contract that stays while implementations change. Check four things on each component surface. Detail and a worked example: `references/components.md` § High-level API design checks.

1. **Passed in, not built inside.** Clients, stores, file formats and consumers are built at the app edge and passed to the component. Prefer an **enum** when this crate owns every variant (formats, storage providers, a `#[cfg(test)]` fake); keep a **trait** only for an open set or implementations from another crate. Use `Arc` only for values several tasks share.  
2. **Easy to test.** Each component is built in tests through its real constructor plus `#[cfg(test)]` fakes; time is a parameter, not a clock read inside; tests use the public API, so swapping an implementation does not rewrite them.  
3. **Performance named up front.** For each unit of work, say the hot path and its cost: allocations, locks or atomics per record; CPU-heavy work on async runtime threads (move it off); memory per unit, bookkeeping included; backpressure counted in the resource that runs out (usually bytes); work units small enough to run in parallel.  
4. **Change stays inside.** List what changes when an implementation is swapped (storage provider, file format, single vs multipart upload). The public API must not expose implementation units such as upload slots, parts or files.  

When a component API is in scope, report `api-design: ok` (one line per check) or findings with path:line.

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
- Treat “no layers” as “no public use-case orchestration on the component”  
- Demand Cosmic/DDD folder sets or formal UoW/repository types by default  
- Treat this crate’s `types` module as a god-module, or demand one workspace-wide `types` crate for unrelated jobs  

## Output section title

`## 2. Architecture`
