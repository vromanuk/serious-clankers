# Component-based architecture (project layout)

**This pack’s default** for how to lay out projects and components — when **building** and when **reviewing**. Not a special-case style.

**Core idea:** a component owns one **job**, exposes a **small public interface** at its boundary, and keeps everything else **private**. Outsiders depend on that interface only.

**Public interface:** prefer **one public struct** for the job; its **methods** are the use cases — they **orchestrate** once (read → decide → write). Hold collaborators on the struct. Not a bag of stray helpers (or free `pub fn`s that re-pass the same deps). Not app-wide `services/` folders. “Service layer” in other books ≈ **this struct**, nothing more.

**Interior:** pure rules (**thinking** / sans-IO) + thin **shell** for IO. Depth: skill **`design-components`** → `references/guide.md` and `skeptic-testability` → `pure-core.md`.

**Not required:** directories literally named `api/` or `internal/`. Those names are one optional convention (common in Java so tools like ArchUnit can scan package paths). In Rust, the natural form is often:

```text
billing/
  mod.rs          # public interface: job struct + pub use of types callers name
  types.rs        # public data types (or types/mod.rs). Not the job struct.
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

### Layers ban ≠ “no public orchestration”

| Avoid (project spine) | Keep (inside a component) |
|----------------------|---------------------------|
| App-wide folders: `controllers/`, `services/`, `repositories/` for every feature | **One job struct** + use-case **methods** (`allocation.allocate`, `.add_batch`) |
| Technical layers as the only structure | Private rules + store **under** the job |
| Exporting every helper so callers orchestrate | **Methods** on the public struct orchestrate once |

The banned idea is **horizontal ownership soup**. The sound idea is a **deep public face** that runs the workflow once, with pure decisions testable without IO. See `design-components/references/guide.md`.

---

## Rules (always)

1. **One component = one job + one namespace** (package, module tree, or crate).  
2. **Public surface vs private interior** — outside code may depend on the surface only.  
3. **Public surface = one job struct + use-case methods** — not a re-export of every private step (`load_*`, `validate_*`, `write_*` sequenced by the caller). Prefer methods over free public orchestration functions that re-pass the same deps.  
4. **Orchestration once** — load → check → decide → save lives in the method (or private helpers only it calls), not reimplemented in each HTTP/CLI handler.  
5. **Interior may nest helpers / sub-parts** (private modules under the component). “One function per task” applies **inside**; most tasks stay **unexported**. Pure thinking may stay free functions **privately**.  
6. **Depend only on other components’ public surfaces** — never dig into another job’s private modules.  
7. **Data ownership** — one owner writes a given set of facts/tables; other components go through the owner’s API.  
8. **Compose at the edge** — app/binary/server wires components (construct the struct, pass it around); components do not form a free-for-all mesh.  
9. **Prefer enforceable structure** — language visibility first; optional automated rules when useful.  
10. **Deep over shallow** — simple interface relative to power; pull complexity down into the component; hide design decisions; do not export a temporal step pipeline. Detail: `design-components/references/philosophy-of-design.md` (Ousterhout).  

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
// billing/mod.rs — public face = job struct + methods (+ types callers need)
mod rules;   // pure thinking
mod store;   // private IO

pub struct Billing {
    store: store::Store,
}

impl Billing {
    pub fn new(store: store::Store) -> Self {
        Self { store }
    }

    pub fn issue_invoice(&mut self, /* … */) -> Result<Invoice, IssueInvoiceError> {
        // orchestrate: load → pure rules → save
        todo!()
    }
}

pub use types::Invoice; // definition lives in types, not beside the shell
// store and step helpers stay private
```

```rust
// outside — only the surface
use crate::billing::{Billing, Invoice};

let mut billing = Billing::new(store);
billing.issue_invoice(/* … */)?;

// not ok: use crate::billing::store::…;  // private path of another job
// not ok: public re-exports of load_rows / validate_line / write_row for callers to sequence
```

**Good enough:** one file component (`billing.rs`) with a small public struct and private items, if the job is small — still prefer **struct + methods** over a flat `pub` helper list.

**Also fine:** multi-crate (`billing` crate, `check_engine` crate) when the boundary must be hard — same idea: crate root is the public interface.

### Public data types

A public **data** type callers name — `pub struct`, `pub enum`, `pub type` — is defined in that crate’s `types` module (`types.rs`, or `types/mod.rs` and files under it). If the crate has no `types` module, add one and move the type there. The crate root may `pub use` it so the short path stays.

The definition site is what counts. `pub struct KafkaRecord` in `processing/listener.rs`, then `pub use` from `lib.rs`, is still outside `types`.

| Stays out of `types` | Why |
|----------------------|-----|
| The job struct (`KafkaConsumer`, `KafkaClient`, `Billing`) | It is the component. Its methods are the use cases. |
| A trait that is the behavior (`RecordProcessor`) | It is what the caller implements, not a data shape. |
| `pub(crate)` and private types (`BrokerRecord`) | Only this crate uses them. They stay next to that code. |
| A one-file script with no library surface | No module split for a one-shot. |

This is one crate’s public data, not a workspace-wide types crate. Mixing unrelated jobs into one shared types crate is still the god-module smell.

---

### One test seam for a concrete collaborator

**First principle:** a type should exist for a job. When a type exists only so tests can replace one collaborator, the seam is the collaborator, not a second copy of the component.

The common shape it replaces: the external client is a concrete type (generated gRPC client, SDK client) with no trait to fake. So a private trait is added for it, the rules move into a generic inner struct, and the public job struct only fixes the type parameter and forwards every call.

**Before — two types and a trait, for one fake:**

```rust
pub struct AssetClient {
    rpc_caller: AssetRpcCaller<AssetApiClient<TraceableChannel>>, // only fixes R
}

impl AssetClient {
    pub async fn get_hierarchy_descendants(&self, /* … */) -> Result<Vec<Row>, AssetError> {
        self.rpc_caller.get_hierarchy_descendants(/* … */).await // forwards
    }
}

trait AssetRpc: Clone + Send + Sync { /* one method per RPC */ } // one production impl

struct AssetRpcCaller<R> { asset_rpc: R, retry_budget: Arc<RetryBudget>, /* … */ }

impl<R: AssetRpc> AssetRpcCaller<R> { /* the real rules: request, retry, deadline, metrics */ }

// tests.rs
impl AssetRpc for FakeAssetRpc { /* … */ }
let client = AssetRpcCaller::new(fake, /* … */); // tests a private type
```

**After — one job struct, one seam:**

```rust
pub struct AssetClient {
    transport: Transport,
    retry_budget: Arc<RetryBudget>,
    call_deadline: Duration,
}

impl AssetClient {
    pub async fn get_hierarchy_descendants(&self, /* … */) -> Result<Vec<Row>, AssetError> {
        // the real rules live here, once
    }
}

/// Where one attempt goes: Asset over gRPC, or a fake in tests.
///
/// The generated client is a concrete type with no trait to fake, so the
/// client's rules run against this enum and tests swap in the fake.
#[derive(Clone)]
enum Transport {
    Grpc(AssetApiClient<TraceableChannel>),
    #[cfg(test)]
    Fake(tests::FakeAssetRpc),
}

impl Transport {
    async fn get_hierarchy_descendants(&mut self, request: Request<Req>) -> Result<Vec<Resp>, Status> {
        match self {
            Self::Grpc(client) => client.get_hierarchy_descendants(request).await?.into_inner().try_collect().await,
            #[cfg(test)]
            Self::Fake(fake) => fake.get_hierarchy_descendants(request).await,
        }
    }
}

// tests.rs — the real job struct, its public method
let client = AssetClient::new(Transport::Fake(fake), /* … */);
client.get_hierarchy_descendants(/* … */).await;
```

**What it buys:** one fewer type, no trait, no type parameter, and tests that run the public method instead of a private copy of it. The seam sits at the one thing tests actually replace.

**What it costs:** the production enum names a test-only variant. Test builds may trip `clippy::large_enum_variant` (a big real client next to a small fake). Allow it for test builds only, with the reason — do not box the real client, which would allocate on every clone in production:

```rust
#[cfg_attr(test, expect(clippy::large_enum_variant, reason = "only test builds have the small Fake variant"))]
```

**Keep the trait when:**

| Situation | Why the trait is a real seam |
|-----------|------------------------------|
| Two or more production implementations (object store vs local disk) | Production chooses between them; the enum would not be test-only |
| Another crate must supply the fake (a `test-util` feature for callers' tests) | `#[cfg(test)]` code is visible only inside this crate |
| The collaborator is the component's public extension point (`RecordProcessor`) | Callers implement it; it is the behavior, not a test hook |

**Review flag:** “`AssetRpcCaller<R>` and trait `AssetRpc` exist only for the test fake; the public `AssetClient` only forwards. Fold the rules into `AssetClient` with a private `Transport` enum and a `#[cfg(test)]` fake variant.”

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
| **Many public step-helpers; callers reassemble the workflow** | Shallow component; missing use-case face |
| **Free public orchestration fns that re-pass the same deps** | Prefer a job struct that holds collaborators |
| **HTTP/CLI contains load→decide→save for a multi-step job** | Orchestration should sit once on the component struct |
| **Business rules only inside SQL/HTTP with no pure core** | Hard to test; split thinking vs shell |
| Two jobs write the same tables | Hidden coupling; hard to split |
| Component A reaches through B to B’s private dependency | Skip levels; use the public surface |
| “Common” bag of domain types for everything | Often a proto-god-module. One crate’s own `types` module is not that — it is where **this** crate’s public data types go. A workspace-wide types crate that mixes unrelated jobs still is |
| **Public struct/enum defined outside `types`** (`KafkaRecord` in `listener.rs`, then `pub use` from the crate root) | Callers cannot find the data type next to the others. Move the definition into `types.rs` or `types/`. Re-export from the root if the short path should stay |
| New package theater for a one-shot script | Keep local |
| **Private trait + generic inner struct only for a test fake; public struct only forwards** | Two types and a trait for one seam; tests hit a private copy. Use one job struct and a private enum with a `#[cfg(test)]` fake variant (§ One test seam) |
| **Missing `api/` folder** when `mod.rs` already is the surface | **Not a smell** — do not flag |
| **Having a private `service.rs` use-case module inside a job** | **Not a smell** — that *is* the service layer idea |

| Good signal | Why |
|-------------|-----|
| Namespace named for the **job** | Clear owner |
| One public **job struct** + few methods (+ needed types); private modules underneath | Deep face; deps in one place |
| Edge binary constructs the struct and stays thin | No cyclic private mesh |
| One owner for a write-set of data | Decompose-friendly |
| Pure rules unit-tested; IO in thin shell | Sans-IO / thinking vs shell |

---

## Review one-liners

- `components: ok` — e.g. “billing exports `Billing` with `issue_invoice`; store private; handlers call methods; no cross-private imports.”  
- Finding: “handler uses `other_job::store::…` — use the public surface or move code.”  
- Finding: “layout is controllers/services/repos only; no job-shaped owner for this change.”  
- Finding: “two crates write `orders` rows — pick one owner.”  
- Finding: “root re-exports `load_*` / `validate_*` / `write_*`; callers sequence them — fold into methods on a job struct.”  
- Finding: “allocate workflow only lives in the HTTP handler — move orchestration onto the component struct.”  
- Finding: “several `pub fn`s all take the same store — hold it on a public struct.”  
- Finding: “`KafkaRecord` is declared in `processing/listener.rs` and only re-exported. Move it into `types`. Add `types.rs` or `types/mod.rs` if the crate has none.”  
- Not a finding: “no `internal/` directory.”  
- Not a finding: “`KafkaConsumer` stays at the component root” — that is the job struct, not a data type.  
- Not a finding: “`BrokerRecord` is `pub(crate)` next to the shell that builds it.”  
- Not a finding: “pure rules are free functions inside the component.”  

---

## High-level API design checks

The public API of a component is what callers and tests depend on. Implementations behind it will change (another storage provider, another file format, multipart instead of one PUT). Design the API so those changes stay inside.

| Check | Ask | Good | Flag |
|-------|-----|------|------|
| Passed in | Does the component build its own clients, stores or formats? | built at the app edge and passed to the constructor | `S3Client::new()` inside a component; a format chosen by `if` deep inside |
| Enum or trait | Who owns the implementations? | enum when this crate owns every variant; trait for an open set or another crate's implementations | `Arc<dyn Trait>` with one production implementation and one fake |
| `Arc` | Is the value shared across tasks? | `Arc` (or a cheap-clone handle) only when shared; move ownership otherwise | `Arc` around values one task owns |
| Tests | Can tests build the real component and swap only the collaborator? | real constructor + `#[cfg(test)]` fake; time as a parameter | tests that need a broker, a clock read, or private helpers to reach the logic |
| Hot path | What runs per record, and what does it cost? | no allocation or lock per record where avoidable; work per batch | per-record locks, allocations, channel sends or metric updates on a high-rate path |
| Runtime threads | Does CPU-heavy work run on async runtime threads? | `spawn_blocking` for heavy work (encoding, compression) | seconds of CPU inside an async task that shares the runtime with I/O loops |
| Memory | What does one unit of work hold, bookkeeping included? | counted in the limit that pauses input | per-record ids or tracker entries left out of the memory limit |
| Backpressure | What resource runs out first? | a limit in that resource (usually bytes); retries do not hold slots | slot counts that tie memory and concurrency together; a retry that holds a worker while it sleeps |
| Parallel work | Can the work be split for idle workers to pick up? | units small enough to spread (parts, files) | one huge unit no idle worker can share |
| Containment | What changes when an implementation is swapped? | only the component's interior | public API exposing upload slots, parts, files, or provider types |

### Worked example: bulk relay (sketch)

```rust
// The window component: rows in, stored windows out. No files, parts or slots in the API.
pub struct WindowExport { .. }
impl WindowExport {
    pub fn new(settings: WindowExportSettings, file_format: BulkFileFormat,
               bulk_bucket: BulkBucket, acknowledgment: Acknowledgment, cancel: CancellationToken) -> Self;
    pub fn add_rows(&mut self, processing_ms: i64, rows: Vec<(InstanceId, BulkRow)>, ack_id: AckId);
    pub fn close_due_windows(&mut self, processing_ms: i64);
    pub fn can_accept_rows(&self) -> bool;                  // memory limit in bytes
    pub async fn wait_for_stored_window(&mut self) -> StoredWindow;
}

// The format: an enum this crate owns. Parquet is one more variant.
#[derive(Clone, Copy)] pub enum BulkFileFormat { Csv }

// The store: a struct with a private enum; the provider and single vs multipart stay inside.
#[derive(Clone)] pub struct BulkBucket { storage: Storage }
enum Storage { S3 { .. }, #[cfg(test)] Fake(FakeBucket) }
```

What changes when part uploads replace whole-file uploads: the interior of `WindowExport` and additions to `BulkBucket`. The relay loop, the record mapper and the tests written against `WindowExport`'s public API stay.

### Review one-liners

- Finding: "`WindowUploads::try_start_upload` returns `AllUploadersBusy` — the caller now manages upload slots; keep slots inside and expose `can_accept_rows()`."
- Finding: "`Chunks<P: PutObject, R>` — a trait and a type parameter for one production store; pass a `BulkBucket` with a private enum fake."
- Finding: "each window keeps one `AckId` per record and the memory limit ignores it — count it, or keep ranges."
- Finding: "CSV for a 256 MiB file is encoded inside an async task — move it to `spawn_blocking` or encode as rows arrive."
- Not a finding: "`Arc<dyn KeyFilter>` — another crate's API asks for it."

---

## Not this file

- Merged interior design (public face + pure-core/shell, short guide) → **`design-components/references/guide.md`**  
- Type-level “caller cannot forget X” → `type-driven.md`  
- Misuse-resistant Rust API craft → `rust-apis.md`  
- Pure vs IO placement samples → skeptic-testability `pure-core.md`  
- Absolute ban IDs → skeptic-hard-rules (`HR-private-import`)
