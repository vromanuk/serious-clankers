# Component-based architecture (project layout)

**This pack’s default** for how to lay out projects and components — when **building** and when **reviewing**. Not a special-case style.

**Core idea:** a component owns one **job**, exposes a **small public interface** at its boundary, and keeps everything else **private**. Outsiders depend on that interface only.

**Not required:** directories literally named `api/` or `internal/`. Those names are one optional convention (common in Java so tools like ArchUnit can scan package paths). In Rust, the natural form is often:

```text
billing/
  mod.rs          # public interface: pub use / pub fn / pub struct that outsiders may use
  invoice.rs      # private modules (not re-exported)
  store.rs
  batch.rs
```

or a crate root `lib.rs` that re-exports only the public surface.

**Sources (original packaging examples — optional folder names):**

- Talk: [Let’s build components, not layers](https://www.youtube.com/watch?v=-VmhytwBZVs) (Tom Hombergs, Spring I/O 2019)  
- Article: [Clean Architecture Boundaries with Spring Boot and ArchUnit](https://reflectoring.io/java-components-clean-boundaries/)  
- Example repo: [thombergs/components-example](https://github.com/thombergs/components-example) (`check-engine`)

Always-on short form: pack `context/AGENTS.md` § Project layout.

### When building (default)

Unless the user or an existing tree requires another shape:

1. Place new work in a **job-named** component (create one if the job is new).  
2. Define the **public contract at the component root** (`mod.rs` / `lib.rs` / package exports) — only what outsiders need.  
3. Keep implementation in **private** modules (do not re-export them).  
4. Wire components from the **app/binary edge**, not via private cross-imports.  
5. Do not introduce app-wide layer folders as the primary layout for multi-job code.  
6. Tiny scripts/tools: keep local; do not invent a component graph for a one-shot.  
7. Do **not** invent `api/` / `internal/` folders just to match a blog diagram if the root already is the public interface.

---

## Why components, not layers

Horizontal layers (`controller` / `service` / `repository` across the whole app) make every feature touch every layer and **blur ownership**. Over time, “anything may depend on anything.”

A **component** owns one **job** end to end (namespace named for that job). Other code may use only its **public API**. Internals stay private so changes stay local and decomposing later is cheaper.

Clean dependencies → understandability, maintainability, extensibility, decomposability.

---

## Rules (always)

1. **One component = one job + one namespace** (package, module tree, or crate).  
2. **Public surface vs private interior** — outside code may depend on the surface only.  
3. **Interior may nest helpers / sub-parts** (private modules under the component).  
4. **Depend only on other components’ public surfaces** — never dig into another job’s private modules.  
5. **Data ownership** — one owner writes a given set of facts/tables; other components go through the owner’s API.  
6. **Compose at the edge** — app/binary/server wires components; components do not form a free-for-all mesh.  
7. **Prefer enforceable structure** — language visibility first; optional automated rules when useful.

The rule is the **boundary**, not a particular folder spelling.

---

## Preferred shape (Rust-first): root defines the public interface

```text
src/
  billing/
    mod.rs           # ← public surface for the billing job
    invoice.rs       # private unless re-exported from mod.rs
    store.rs
    batch.rs
  check_engine/
    mod.rs           # ← public surface
    queue.rs
    db.rs
    runner.rs
  main.rs            # wires components; uses billing::… and check_engine::… only via their roots
```

```rust
// billing/mod.rs — the public interface
mod invoice;
mod store;
mod batch;

pub use invoice::{Invoice, InvoiceCalculator};
// store, batch stay private to this module tree
```

```rust
// outside — only the surface
use crate::billing::{Invoice, InvoiceCalculator};

// not ok: use crate::billing::store::…;  // private path of another job
```

**Good enough:** one file component (`billing.rs`) with private items unexported, if the job is small.

**Also fine:** multi-crate (`billing` crate, `check_engine` crate) when the boundary must be hard — same idea: crate root is the public interface.

---

## Optional convention: folders named `api` / `internal`

Some codebases (Hombergs / reflectoring Java examples) use explicit packages:

```text
billing/
├── api/           # public — outsiders may use this
└── internal/      # private — only this component
    ├── batchjob/
    └── database/
```

Useful when:

- the language has weak privacy across subpackages (classic Java), or  
- you want a **path-based** rule for ArchUnit / similar (“nothing outside `*.internal` may depend on it”).

In Rust this is **optional**. Prefer root `mod.rs` / `lib.rs` as the surface unless the team already uses `api`/`internal` names or needs path-scanned rules.

If you do use the names, same ownership rules apply; nested sub-parts stay under the private side.

### Nested dependency inversion (language-agnostic)

Inside a component, prefer: **callers depend on small interfaces/types; storage/adapters implement them** — not the reverse. Swap DB tech without rewriting the job’s public API.

### App / server edge

```text
main / server / app
  → depends on each component’s public surface only
  → wires startup, config, DI

components (modules or crates)
  billing, check_engine, …
```

---

## Worked sketch — check-engine (ideas, not folder dogma)

From [components-example](https://github.com/thombergs/components-example) (illustrates jobs; their tree uses `api`/`internal` for Java tooling):

**Job:** run arbitrary checks against a web page.

**Public surface (only this for outsiders):**

- Schedule a check (async)  
- Query results  

**Private parts (implementation of that job):**

| Part | Role |
|------|------|
| queue | schedule path; talks to a queue |
| database | query path; owns check-result tables |
| checkrunner | runs work; stores results |

**Data rule:** only the database part of this job touches those tables.

**Enforcement idea:** outsiders must not depend on private modules of the component. In Java examples that is often “no access into `*.internal` packages.” In Rust, **private modules + no re-export** is usually enough.

---

## What to flag in review

| Smell | Why |
|-------|-----|
| Feature split only by layer (`…/service`, `…/repo` for the whole app) | No job owner; deps sprawl |
| Outside code imports another job’s private module / path | Broken encapsulation (`HR-private-import`) |
| Public root re-exports everything (no real surface) | Boundary is fake |
| Two jobs write the same tables | Hidden coupling; hard to split |
| Component A reaches through B to B’s private dependency | Skip levels; use the public surface |
| “Common” bag of domain types for everything | Often a proto-god-module |
| New package theater for a one-shot script | Keep local |
| **Missing `api/` folder** when `mod.rs` already is the surface | **Not a smell** — do not flag |

| Good signal | Why |
|-------------|-----|
| Namespace named for the **job** | Clear owner |
| Thin public root; private modules underneath | Change stays local |
| Edge binary wires components | No cyclic private mesh |
| One owner for a write-set of data | Decompose-friendly |

---

## Review one-liners

- `components: ok` — e.g. “billing root re-exports InvoiceCalculator only; store/batch private; no cross-private imports.”  
- Finding: “handler uses `other_job::store::…` — use the public surface or move code.”  
- Finding: “layout is controllers/services/repos only; no job-shaped owner for this change.”  
- Finding: “two crates write `orders` rows — pick one owner.”  
- Not a finding: “no `internal/` directory.”  

---

## Not this file

- Type-level “caller cannot forget X” → `type-driven.md`  
- Misuse-resistant Rust API craft → `rust-apis.md`  
- Pure vs IO placement → skeptic-testability  
- Absolute ban IDs → skeptic-hard-rules (`HR-private-import`)
