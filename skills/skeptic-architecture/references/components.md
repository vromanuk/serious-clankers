# Component-based architecture (project layout)

**This pack’s default** for how to lay out projects and components — when **building** and when **reviewing**. Not a special-case style.

**Portable rules** for job-shaped components with a clear public surface and a private interior.  
Java/Kotlin packaging is the classic illustration; the **jobs and dependency rules** apply in Rust (modules/crates), Go, TypeScript, etc.

**Sources (load these if you need the original):**

- Talk: [Let’s build components, not layers](https://www.youtube.com/watch?v=-VmhytwBZVs) (Tom Hombergs, Spring I/O 2019)  
- Article: [Clean Architecture Boundaries with Spring Boot and ArchUnit](https://reflectoring.io/java-components-clean-boundaries/)  
- Example repo: [thombergs/components-example](https://github.com/thombergs/components-example) (`check-engine`)

Always-on short form: pack `context/AGENTS.md` § Project layout. This file is the **depth + examples**. Not a Spring tutorial.

### When building (default)

Unless the user or an existing tree requires another shape:

1. Place new work in a **job-named** component (create one if the job is new).  
2. Put only the **public contract** on the surface (`api` / `pub` re-exports).  
3. Keep implementation and sub-components under **private** (`internal` / private modules).  
4. Wire components from the **app/binary edge**, not via internal cross-imports.  
5. Do not introduce app-wide layer folders as the primary layout for multi-job code.  
6. Tiny scripts/tools: keep local; do not invent a component graph for a one-shot.

---

## Why components, not layers

Horizontal layers (`controller` / `service` / `repository` across the whole app) make every feature touch every layer and **blur ownership**. Over time, “anything may depend on anything.”

A **component** owns one **job** end to end (namespace named for that job). Other code may use only its **public API**. Internals stay private so changes stay local and decomposing later (extract a service, split a crate) is cheaper.

Clean dependencies → understandability, maintainability, extensibility, decomposability.

---

## Rules (always)

1. **One component = one job + one namespace** (package, module tree, or crate).  
2. **Public surface vs private interior** — outside code may depend on the surface only.  
3. **Interior may nest sub-components** (each with its own namespace under the parent’s private area).  
4. **Depend only on other components’ public surfaces** — never reach into `internal` / private modules of another job.  
5. **Data ownership** — one owner writes a given set of facts/tables; other components go through the owner’s API (not shared “god” tables across jobs).  
6. **Compose at the edge** — app/binary/server wires components; components do not form a free-for-all mesh.  
7. **Prefer enforceable structure** — visibility + one automated rule when possible (see below).

Language visibility alone is not enough when you need **sub-packages/modules** inside a component (public helpers for siblings would leak to the whole world). Structure + tools fix that.

---

## Layout shape (api / internal)

Canonical package shape (Hombergs / reflectoring):

```text
billing/                    # component: job = billing
├── api/                    # public — outsiders may use this
└── internal/               # private — only this component (and its subtrees)
    ├── batchjob/
    │   └── internal/
    ├── database/
    │   ├── api/            # API for *siblings under billing.internal*, not the world
    │   └── internal/
    └── …                   # implementation of top-level api
```

| Path | Who may use it |
|------|----------------|
| `…/api` | Outside the enclosing component (and sibling code that is allowed to see that surface) |
| `…/internal` | Only code under that `internal` tree (including nested sub-components) |

**Sub-components** live under the parent’s `internal` so they stay hidden from the outside world even if they expose their own nested `api` for local wiring.

### Nested dependency inversion

Inside a component, invert deps so **internals implement APIs**, not the reverse:

```text
database/
├── api/          # ReadLineItems, WriteLineItems, LineItem  (interfaces / types)
└── internal/     # BillingDatabase implements those APIs
```

Callers (invoice calculator, batch job) depend on **api types**, not on DB technology. Swap storage without rewriting callers.

A sub-component may expose **no** public API at all (e.g. a batch job that only runs on a schedule and calls `WriteLineItems`).

### App / server edge

```text
server/ (or main binary)
  → depends on component public APIs only
  → wires configs / DI / startup

components/
  check-engine/
  billing/
  …
```

Optional: one **crate/module per component** for harder boundaries; same rules work in a single module if namespaces and visibility stay clean. Prefer “structure first”; multi-crate when the boundary must be hard.

---

## Worked sketch — check-engine

From [components-example](https://github.com/thombergs/components-example):

**Job:** run arbitrary checks against a web page (HTML validity, SEO tags, …).

**Public API (only this for outsiders):**

- Schedule a check (async) — e.g. `CheckScheduler`  
- Query results — e.g. `CheckQueries`

**Sub-components under `internal`:**

| Sub-component | Role |
|---------------|------|
| `queue` | Implements schedule API; talks to a queue |
| `database` | Implements query API; owns check-result tables |
| `checkrunner` | Runs scheduled work; stores results; no parent API of its own |

```text
check-engine
├── api/                 # CheckScheduler, CheckQueries, …
└── internal/
    ├── queue/…
    ├── database/…       # only path that touches check tables
    └── checkrunner/…
```

**Data rule:** only the database sub-component touches those tables. Other jobs do not “just join” them.

**Enforcement (example):** one ArchUnit-style rule:

> No class **outside** an `internal` package may depend on a class **inside** that `internal` package.

In the example repo this is a single test that discovers all `*.internal` packages and fails on illegal edges — new components are covered automatically when they follow the naming convention.

---

## Rust mapping (same jobs)

| Java / package idea | Rust shape |
|---------------------|------------|
| Component package | Crate, or top-level module tree for one job |
| `api` | `pub` items re-exported as the crate/module surface |
| `internal` | Private modules; do not `pub use` them from the root for outsiders |
| Nested sub-component | Child modules under the component; still not part of the public re-export |
| Cross-component private import | `other_crate::internal::…` or `other_mod::private_path` from outside → **flag** |
| Compose at edge | `main` / binary / thin app crate depends on component crates and wires them |
| ArchUnit internal rule | Visibility + optional CI (module graph / deny private paths); hard-rule `HR-private-import` |

```text
// Good: crate check_engine
// lib.rs
pub mod api;           // or re-export only api::*
mod internal;          // private

// outside:
use check_engine::api::CheckScheduler;   // ok
// use check_engine::internal::…;       // not ok
```

```text
// Bad: layer folders across jobs
src/
  controllers/
  services/
  repositories/     // every feature bleeds through the same three buckets
```

Prefer:

```text
src/
  billing/
    api.rs | mod api
    internal/…
  check_engine/
    api/
    internal/…
  main.rs           // wires components
```

Visibility in Rust is stronger than Java package-private for submodules — **use it**. Still name the public surface so reviewers and tools can see the boundary without reading every file.

---

## What to flag in review

| Smell | Why |
|-------|-----|
| Feature split only by layer (`…/service`, `…/repo` for the whole app) | No job owner; deps sprawl |
| Outside code imports `…internal…` / private module of another job | Broken encapsulation (`HR-private-import`) |
| Two jobs write the same tables / shared “utility” persistence of domain facts | Hidden coupling; hard to split |
| Public type that is only an implementation detail | Surface too large; harden or move inside |
| Component A reaches through B to B’s dependency | Skip levels; depend on the API you need |
| “Common” / “shared” bag of domain types for everything | Often a proto-god-module; prefer owned types + small shared kernels |
| New package theater for a one-shot script | Keep local; no fake component graph |

| Good signal | Why |
|-------------|-----|
| Namespace named for the **job** | Clear owner |
| Thin public API; fat private tree | Change stays local |
| Sub-components under private parent | Nest without leaking |
| Edge binary wires components | No cyclic component mesh |
| One owner for a write-set of data | Decompose-friendly |

---

## Review one-liners

- `components: ok` — e.g. “billing only exposes invoice calculator; batch + DB under internal; no cross-internal imports.”  
- Finding: “handler imports `other_job::internal::…` — use public API or move code.”  
- Finding: “layout is controllers/services/repos only; no job-shaped owner for this change.”  
- Finding: “two crates write `orders` rows — pick one owner.”  

---

## Not this file

- Type-level “caller cannot forget X” → `type-driven.md`  
- Misuse-resistant Rust API craft → `rust-apis.md`  
- Pure vs IO placement → skeptic-testability  
- Absolute ban IDs → skeptic-hard-rules (`HR-private-import`)
