# Designing components (simple merge)

One architecture story. Not two books side by side.

**Spine:** job-shaped **components** (public surface + private interior).  
**Face:** one public **struct** (the job’s handle) whose **methods** are the use cases — they **orchestrate** once. Hold collaborators on the struct; do not thread the same deps through free functions at every call site.  
**Inside:** **thinking code** (pure / sans-IO) for decisions; thin **shell** for IO.

That is the whole model. Cosmic Python’s “service layer” is just a name for the **public face of a component**. We do **not** adopt their full DDD pattern stack (repository theater, UoW types, aggregates) as default.

Layout rules (folders, Hombergs, no app-wide layers): `skeptic-architecture/references/components.md`.  
Thinking vs shell detail: `skeptic-testability/references/pure-core.md`.

---

## One picture

```text
  HTTP / CLI / main / other components
           |
           |  construct once, call methods
           v
  +--------------------------------------------+
  |  COMPONENT (one job)                       |
  |                                            |
  |  PUBLIC:  struct Allocation { … deps }     |  ← one handle for the job
  |           .allocate(...)                   |  ← use cases (methods)
  |           .add_batch(...)                  |
  |           (+ types/errors callers need)    |
  |                                            |
  |  PRIVATE:                                  |
  |    thinking  — pure rules, decisions       |  ← no disk/net/clock
  |    shell bits — load/save, SQL, HTTP out   |  ← IO at the edge of the job
  |    helpers   — split for clarity, not pub  |
  +--------------------------------------------+
```

**App-wide layers stay banned:**

```text
src/controllers/   src/services/   src/repositories/   ← not the project spine
```

**“Service” means the component’s public API**, not a global `services/` package.

---

## Three rules

### 1. One job → one component → one public struct

- Namespace named for the **job** (`billing`, `allocation`, …).  
- Prefer **one public struct** named for the job (or a clear handle: `Billing`, `AllocationService` only if the plain job name collides).  
- **Methods** on that struct are the use cases (`issue_invoice`, `allocate`) — not a chain of free step-helpers.  
- **Collaborators** (store, clock, clients) live **on the struct** (or behind a small private field), constructed at the app edge once.  
- Private modules hold the rest. Language visibility first (Rust: don’t re-export).

Same as `components.md`. This guide stresses: the face is a **struct + methods**, not a bag of free functions and not a helper dump.

Free functions are OK for **pure** thinking code (private or carefully shared) and for tiny one-file tools. They are **not** the default public orchestration surface of a multi-step job.

### 2. Methods orchestrate once

Each public method owns the story:

```text
read what you need → decide (prefer pure) → write effects → return a simple result
```

Callers (HTTP handler, CLI, another component) must **not** reassemble that sequence from ten public helpers.

| Bad (shallow / stray) | Good (deep face) |
|----------------------|------------------|
| `pub load_…`, `pub validate_…`, `pub write_…` | `allocation.allocate(...)` does those **privately** |
| Handler contains the whole workflow | Handler calls `allocation.allocate(...)` only |
| Free `pub fn allocate(store, clock, …)` every call | `Allocation { store, clock }` + `.allocate(…)` |
| Same 15-line load/check/save copied twice | One method; both call sites use the same handle |

**Deep module** (Ousterhout, short form): simple interface, powerful body. Pull complexity **down** into the component so the call site stays easy. Small private functions for clarity are fine; **exporting** every step is not.

### 3. Interior: thinking vs shell (sans-IO)

| Piece | Job | IO? |
|-------|-----|-----|
| **Thinking code** | Rules and decisions: data in → decision/value out | **No** network, disk, env, clock, random, threads |
| **Shell** (inside the component or at app edge) | Load inputs, call thinking, apply effects | **Yes** |

Sans-IO / pure-core idea: the interesting logic is testable with fixed inputs. Time is a **parameter**, not `now()` inside the core.

```rust
// Thinking — pure (free function or private module is fine)
enum Plan { DoNothing, Update(Customer), UpdateAndEmail(Customer, String) }

fn plan_update(existing: &Customer, new: &Customer) -> Plan {
    // no db, no mail, no Instant::now
    todo!()
}

// Public face — struct holds collaborators; methods orchestrate
pub struct Customers {
    db: Db,
    mail: Mail,
}

impl Customers {
    pub fn apply_update(&self, id: Id) -> Result<(), Error> {
        let existing = self.db.read(id)?;
        let incoming = self.db.read_incoming(id)?;
        match plan_update(&existing, &incoming) {
            Plan::DoNothing => Ok(()),
            Plan::Update(c) => self.db.write(c),
            Plan::UpdateAndEmail(c, msg) => {
                self.db.write(c)?;
                self.mail.send(&msg)?;
                Ok(())
            }
        }
    }
}
```

You do **not** need a trait named Repository or UnitOfWork for this shape. Store concrete collaborators on the struct, or a small trait field **when** you need a fake for tests — only then.

Full pure-core samples: `skeptic-testability/references/pure-core.md`.

---

## How the pieces fit (plain words)

| Idea | In this pack |
|------|----------------|
| Hombergs “components not layers” | Project spine = jobs, not technical tiers |
| Cosmic “service layer” | **Public struct + use-case methods** — one sentence, not a framework |
| Sans-IO / pure core / shell | **How the interior is written** so rules are testable |
| Ousterhout deep modules | Public face stays small; hide the messy steps |
| SOLID | Optional checklist: one job, small ports if any, IO not inside pure rules |

We **do not** require Cosmic’s default folder set (`domain/`, `service_layer/`, `adapters/`, `entrypoints/`) or their full pattern ladder. Optional ideas only when a real pain appears — see appendix.

---

## Worked sketch (keep it small)

**Job:** allocate stock for an order line.

```text
allocation/
  mod.rs       # PUBLIC: struct Allocation, AllocateError, …
  rules.rs     # PRIVATE thinking: Batch, choose_batch, …
  store.rs     # PRIVATE shell: load/save batches (SQL or in-memory)
```

```rust
// rules.rs — pure
pub(crate) fn choose_batch_ref(line: &OrderLine, batches: &[Batch]) -> Result<String, OutOfStock> {
    // sort / pick / no IO
    todo!()
}

// mod.rs — public face is a struct
pub struct Allocation {
    store: Store, // concrete or Box<dyn …> only if you need it
}

impl Allocation {
    pub fn new(store: Store) -> Self {
        Self { store }
    }

    pub fn allocate(&mut self, order_id: &str, sku: &str, qty: u32) -> Result<String, AllocateError> {
        let line = OrderLine::new(order_id, sku, qty);
        let batches = self.store.list_for_sku(sku)?;
        if batches.is_empty() {
            return Err(AllocateError::InvalidSku(sku.into()));
        }
        let batch_ref = choose_batch_ref(&line, &batches)?;
        self.store.save_allocation(&line, &batch_ref)?;
        Ok(batch_ref)
    }

    pub fn add_batch(&mut self, /* … */) -> Result<(), AllocateError> {
        todo!()
    }
}
```

HTTP stays dumb: hold `Allocation` in app state → parse JSON → `allocation.allocate(...)` → status code.

Tests:

- **Rules** with table tests (no DB).  
- **Use case methods** with an in-memory store on `Allocation { store: fake }` if orchestration matters.  
- **Few** real-DB tests if SQL is non-trivial.

---

## Stray helpers (the failure mode)

Agents often: split “one function per task” → re-export everything → callers become the orchestrator.

**Rule:** one function per task **inside** the component. Public surface is preferably **one struct** whose methods are the use cases.

| Smell | Fix |
|-------|-----|
| Root exports step chain | Methods on the job struct; steps private |
| Free `pub fn`s that all take the same store/clock | Struct fields hold deps; methods use `self` |
| Handler is a mini service layer | Move sequence onto the struct |
| Shared `utils` with domain rules for three jobs | Rules live in the owning component |

---

## Good enough (stop here)

Do **not** add structure by default:

| Skip unless it hurts | Why you might add it later |
|----------------------|----------------------------|
| Formal repository trait hierarchy | Multiple storage backends or painful fakes |
| Unit-of-work type | Multi-step commits / rollback policy scattered |
| Aggregate root jargon | Concurrency / multi-entity invariants need one write boundary |
| App-wide ports-and-adapters folders | Team already works that way; still map to **job** components |

**Good enough:** one public struct with two methods + private pure rules + private store beats a perfect diagram with empty packages.

One-shot scripts: stay local; no component graph.

---

## Design checklist

1. **Job name?** → public **struct** name?  
2. **Public use cases?** (list verbs — those are **methods**)  
3. **What deps live on the struct?**  
4. **Which rules are pure?** (thinking code; pass time in)  
5. **Where is IO?** (methods / private shell — not inside pure rules)  
6. **Can a stranger use only the struct?** (no step order to learn)  
7. **What are we not building yet?**  

---

## Review flags

| Flag | Why |
|------|-----|
| Many public step-helpers | Orchestration leaked |
| Free public orchestration fns that re-pass the same deps | Prefer a struct that holds them |
| Handler owns multi-step load→decide→save | Move onto component struct |
| Rules mixed with SQL in one blob, untestable | Split thinking vs shell |
| Two components write the same tables | Ownership broken |
| Digging into another job’s private modules | Broken boundary |

| Good | Why |
|------|-----|
| One public job struct + few methods | Deep face; deps in one place |
| Pure rules unit-tested without mocks of the world | Sans-IO |
| App constructs the struct and calls methods | Edge composition |

---

## Sources (optional reading — not a second curriculum)

| Source | We take |
|--------|---------|
| Hombergs — [components not layers](https://www.youtube.com/watch?v=-VmhytwBZVs) | Job packages, not app-wide layers |
| Cosmic Python [ch.4 service layer](https://www.cosmicpython.com/book/chapter_04_service_layer.html) | “Use case entrypoint” ≈ our public face — **not** the full book as default |
| Ousterhout — *Philosophy of Software Design* | Deep modules; pull complexity down |
| Tilkov — [“Good Enough” Architecture](https://www.youtube.com/watch?v=nb0Ru40548U) | Enough modularization for the real problem |
| Pack pure-core | Thinking vs shell / sans-IO |

---

## Appendix: Cosmic patterns (only when needed)

If the job grows painful, these map **into** the same component (still not app-wide layers):

| Cosmic name | Simple reading here |
|-------------|---------------------|
| Domain model | Thinking code / pure rules |
| Service layer | Public job struct + use-case methods |
| Repository | Optional store port or thin `store` module |
| Unit of work | Explicit transaction/commit boundary when multi-write |
| Aggregate | One consistency boundary for related writes |

Prefer Cosmic Python’s own trade-off tables before adopting the ceremony. Default remains this short guide + `components.md` + pure-core.

---

## Related

| File | Role |
|------|------|
| `skeptic-architecture/references/components.md` | Layout default, review one-liners |
| `skeptic-testability/references/pure-core.md` | Thinking vs shell |
| `skeptic-conventions/references/write-code.md` | One function per task (interior) |
| `design-components/SKILL.md` | Design workflow |
