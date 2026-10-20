# Comments that explain (why, not only what)

Comments exist for truths **names, types, and tests** do not make cheap: **why this design**, **what benefit**, **why not the obvious alternative**, hazards, ownership, scars.

A fact that is *true but unexplained* is still a weak comment. Stating a rule without the reason leaves the next engineer unable to judge when the rule still applies.

**Google ideas (any language) + review list:** `google-comments.md`  
**Design headers:** `design-headers.md`  
**Module shape (this file):** § Module docs  

---

## The bar: answer “why should I care / why this way?”

| Weak | Strong |
|------|--------|
| States a rule or fact | States the rule **and** the reason or benefit |
| “Do X, not Y” | “Do X, not Y, **because** …; otherwise …” |
| Non-obvious *what* only | Non-obvious *what* **plus** *why it is that way* |
| Narrates the next line | Explains a decision the code cannot show |

Aligned with:

- **Google C++ Comments** — prefer higher-level **why** / intent over restating the next line; self-describing code first.  
- **Ousterhout** (*Philosophy of Software Design*) — comments **reduce complexity**; interface docs are part of the abstraction (what callers must know); implementation comments focus on **why**, not a play-by-play of **how**.  
- **Rust API Guidelines** — examples often show **why someone would use** the item, not only the mechanical *how* to call it ([C-EXAMPLE](https://rust-lang.github.io/api-guidelines/documentation.html)).  
- **rustdoc book** — first line places the crate/module in the ecosystem; then more detail and examples ([How to write documentation](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html)).  
- **PEP 257** — summary line, then elaborate description (we add: elaboration includes **why** when the summary is a design choice).  
- **HTDP design recipe** — state **purpose** (what it does / why it exists for the problem) before implementation; examples make the contract checkable (same spirit as purpose + given/expected).  
- **explain-diff** (this pack) — lead with **intuition**, then reasons and tradeoffs, then mechanical detail.

### Still true (do not over-comment)

- Prefer **clear names and types** over a comment that only supplies a missing verb.  
- Do **not** restate the next line.  
- Dense and useful — not a blog post on every private helper.  
- Short *what* is OK for regex, dense bit/math, odd protocols — still add one *why* when the choice is non-obvious.

---

## Easy to understand at the use site

A reader should get intent **where it’s used** (or on the public `///` / module `//!`) without opening every body.

| On the public item (`//!` / `///`) | Inside the body (`//`) |
|------------------------------------|------------------------|
| Role, **why this shape**, how to use, contracts | Why this branch, order, or skip |
| Why not the obvious alternative (when it matters) | Hazard, “do not change to X” |
| Given/expected when useful | Tricky *how* only when needed |

| Kind | Put | Job |
|------|-----|-----|
| **Module / crate** | `//!` | Intuition + **why** + rules that follow + entrypoints + related + flow when multi-step |
| **Type / function** | `///` | Purpose (**what + why if non-obvious**), contracts, failure |
| **Body** | `//` | Non-obvious choices next to the hard line |

Prefer fixing unclear code over a long comment that apologizes for it.

---

## Module docs (`//!`) — template

A public (or crate-important) module should open with a `//!` block a stranger can use **without reading every item**.

Inspired by explain-diff’s “intuition first, then detail” — but **short** (about half a screen max for a typical module; crate root can be longer).

### Order (use what applies; skip empty sections)

```text
1. One-line role          — where this fits (rustdoc search line; no deep jargon)
2. Intuition               — high-level idea in plain words (toy picture OK)
3. Why / why not           — benefit of this shape; cost of the obvious alternative
4. Rules that follow       — invariants callers must respect (now they make sense)
5. Entrypoints             — start-here names + one phrase each
6. Related                 — sibling modules / out of scope
7. Flow / diagram          — only if multi-step; plain text/ASCII; hazards on the path
```

**Not required:** forced “owns / does not own” template, pure/shell stamps, essay history.

**First line:** one sentence a search/index can show — role of the unit ([rustdoc](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html)).

### Mental check before you ship the `//!`

1. Could a stranger answer “**why is it built this way?**” from the doc alone?  
2. If I deleted the rules and kept only the **why**, would the rules still be recoverable?  
3. Did I only list **what** we do without a **benefit** or a **rejected alternative**?

If (1) or (2) fail, the doc is still a fact sheet — fix it.

---

## Module examples

### Bad — rules without why (your shape)

```rust
//! Clients query views, not raw object keys. One view per data plane (schemas may differ
//! by writer — not one giant union).
```

True, but a new reader does not know:

- why views beat raw keys,  
- what breaks if they key by object id,  
- why one view per plane instead of one mega-schema.

### Good — intuition + why + rules that follow

```rust
//! Read path for historian data planes: clients query **views**, not object-store keys.
//!
//! **Idea:** each data plane exposes a stable query shape (a view). Writers may use
//! different on-disk layouts; the view is the contract clients depend on.
//!
//! **Why views (not raw keys):** keys and file layout change when we re-shard or change
//! codecs. Views keep readers stable so we can evolve storage without rewriting every
//! client. Benefit: cheaper migrations; fewer “wrong key” production bugs.
//!
//! **Why one view per plane (not one giant union):** writers and planes evolve on different
//! schedules. A single union schema forces every reader to understand every writer’s
//! quirks and breaks when any plane adds a field. Separate views isolate change.
//!
//! **Rules:** query only through the plane’s view API; do not construct object keys in
//! callers. Schemas may differ by writer — do not assume one row type across planes.
//!
//! Entrypoints: `open_view`, `query_range`.
//! Cold local profiling is `bench`; stats profiles are `parquet_stats`.
```

### Good — purpose + why boundary + entrypoints (no multi-step flow)

```rust
//! Cold DuckDB read timing and result artifacts for historian-shaped SQL.
//!
//! **Why separate from production paths:** production mixes object-store hydration, gRPC
//! pagination, and connection pools — those dominate latency and hide engine cost.
//! This module isolates DuckDB-cache-cold timing so benches measure the engine, not the fleet.
//!
//! **Caveat:** local "cold" is DuckDB-cache-cold only; OS page cache may still be warm
//! (do not treat local numbers as remote-cold).
//!
//! Entrypoints: `measure_profile`, `write_stats_json`.
//! Out of scope: object-storage hydration, gRPC pagination, production pools.
```

### Good — intuition + flow + hazard

```rust
//! Turn a `stats.json` profile into distribution-matched hive parquet at a chosen scale.
//!
//! **Idea:** treat the profile as a recipe for synthetic series; scale packs while holding
//! per-series rates so downstream queries see realistic cardinality.
//!
//! Values are a seeded random walk per (pack, signal) series (reproducible benches).
//!
//! Entrypoints: `run` (write hive parquet under out_dir), `generate_output_reuse` (suite reuse).
//! Profiling is `parquet_stats`; cold reads are `bench`.
//!
//! ```text
//! GenerateRun::prepare → write_parquet → finish
//! (on failure, partial out_dir is left on disk — remove it yourself before re-running)
//! ```
```

### Flow detail

Prefer **named steps** with outcomes/hazards, not a bare name list:

```rust
//! ```text
//! connect_to_duckdb() → Connection
//!   → resolve parquet files (non-empty)
//!     → measure_profile → ParquetStats
//!       → write_stats_json (file + human summary)
//! ```
```

ASCII is fine. No Mermaid requirement (pack: plain text/ASCII diagrams).

---

## Function / type docs (`///`) — why when non-obvious

**Always:** what it does for the caller (purpose).  
**When the design is a choice:** one phrase of **why** or **why not the obvious alternative**.

```rust
/// Half-open read window from data range and optional span/end.
///
/// Uses end-exclusive bounds so adjacent windows tile without double-counting
/// the boundary sample (the usual bug with inclusive ends).
///
/// given: range min=0 max=200_000_000, no window, no explicit end
/// expected: start=0, end=200_000_001
fn window(...) -> Window { ... }
```

Design headers (purpose + given/expected, no type line): `design-headers.md`.  
Purpose line should not be only a synonym of the function name — include the **problem it solves** or the **non-obvious constraint** when one exists.

---

## Body comments (`//`) — why this branch

```rust
// Workload key joins signals with a unit separator so one signal named "a,b"
// does not collide with two signals "a" and "b". Do not switch to comma join.
fn workload_key(...) -> String { ... }
```

```rust
// CREATE SECRET so keys are redacted in DuckDB errors/settings; map_err so our
// built SQL (which still contains literals) is not attached to the error chain.
conn.execute_batch(&secret)
    .map_err(|_| anyhow::anyhow!("failed to configure object storage credentials"))?;
```

```rust
// Stale writer must not overwrite newer state — generation check is the contract.
```

Ticket without the constraint is weak:

```rust
// Bad
// TICKET-123: skip cache

// Better
// OS page cache not cleared on local cold — use remote for true cold.
// Scar: local warm runs were misread as engine wins (TICKET-123).
```

---

## Scar at the right level

Compat / serde defaults belong on the **type or field** (and a test), not a vague module line:

```rust
/// `_spread` / `distinct_values` default to 0 so older stats.json still load;
/// generate treats 0 as no per-signal variation / no quantization.
#[serde(default)]
pub(crate) frac_unchanged_spread: f64,
```

---

## Settings and numbers

A number in code or config is a decision. The reader needs to know what it limits and why it has this value. The model is HikariCP's pool settings ([README](https://github.com/brettwooldridge/HikariCP#gear-configuration-knobs-baby), e.g. `connectionTimeout`, `maxLifetime`): each says what the setting controls, what happens at the limit, what special values mean, how it relates to other limits, and the default.

### Setting docs (config field, builder method, public constant)

In this order, skipping what does not apply:

1. **What it controls**, from the caller's side: who waits for what, or what is limited.
2. **What happens at the limit**: error returned, item dropped, connection closed.
3. **Special values**: what `0` / `None` means.
4. **Relations and bounds**: settings or outside limits it must stay below or above, and the smallest value that makes sense. A wrong relation fails silently, so this line is often the most valuable ("several seconds shorter than any infrastructure imposed connection time limit", HikariCP `maxLifetime`).
5. **Default**: the value, or a link to the default function/constant that explains it.

Units belong in the type (`Duration`) or the name (`_ms`); that is the naming stage (`HR-api-bare-quantity`). The doc repeats the unit only when neither carries it.

Describe the setting, not the incident that led to it. Replica layouts, deploy order, and outage timelines go in the PR or ticket; the doc states the general behavior the incident exposed, in precise terms ("the TCP handshake does not complete", not "gets no reply").

### Chosen values (defaults, constants, literals)

Next to the value, say why this number: what it is derived from (a measurement, an upstream limit, a protocol constant, a ratio to another setting) and what goes wrong if it is much lower or higher. Provenance alone ("copied from the Scala config", "recommended") is not a reason; keep it only next to one. The reason lives once, at the default; the field doc links there instead of repeating it.

Bad: incident story, vague mechanism, deployment detail in a shared setting:

```rust
/// Longest wait for a new TCP connection to the server to open. When a server restarts, a
/// connection attempt can get no reply at all; without this limit it waits for the OS to give
/// up (about two minutes on Linux) and every request queues behind it until its
/// `request_handler_timeout`, so callers see a long stall instead of a fast failure they can
/// retry, for example on another replica.
pub connect_timeout: Duration,
```

Good: controls → at the limit → default; the reason for the value sits on the default:

```rust
/// Maximum time to wait for a new TCP connection to the server to be established. If it is
/// not established in time, the attempt fails and the requests waiting on it fail with it,
/// instead of waiting until the OS gives up on the connection (about two minutes on Linux).
/// Defaults to [`default_connect_timeout`].
pub connect_timeout: Duration,

/// 1 second. A healthy in-cluster TCP connect takes milliseconds, so this is a wide margin.
/// Keep it well below `request_handler_timeout`, or requests time out before the connect fails.
pub fn default_connect_timeout() -> Duration { Duration::from_secs(1) }
```

Bad: provenance only, so nobody can tell whether 10 s is still right:

```rust
/// Suggested gRPC default keep_alive_timeout value when using nginx
/// Copied from our Scala gRPC configuration: https://…
pub fn default_keep_alive_timeout() -> Duration { Duration::from_secs(10) }
```

Better: state what the 10 s is measured against and what breaks if it changes; keep the Scala link as provenance next to that reason.

---

## Coherence across modules

| Check | Fail when |
|-------|-----------|
| Why present | Module states rules with no reason or benefit |
| Cascade style | Modules mix unrelated flow styles with no pattern |
| Detail level | Sibling private modules: one essay, next empty |
| Entrypoints missing | Multi-entry module with no “start here” names |
| Names match code | Flow still uses old function names |
| Placement | History only on module; `#[serde(default)]` fields have no note |
| No empty templates | Forced “owns / does not own” or pure/shell labels on every doc |

---

## Bad — states the obvious (or unexplained rule)

Same as Google: do not describe what the next line already says. Prefer **why**, or make the code self-describing (see `google-comments.md`).

```rust
// Loop over files
for f in files {
    // Get the name
    let name = f.name();
```

Also bad at module level: only a bare rule list (see “Bad — rules without why” above).

---

## Dense comments (when they help)

Prefer a short block that changes how you **call or change** the code (caller obligations, negative space, policy, **rejected alternative**) over many lines that narrate the algorithm (“// increment i”).

---

## Review checklist (comments stage)

1. Clear English?  
2. Needed at all? (or rename / simplify code)  
3. **Why / benefit / why-not**, not only **what** or a bare rule?  
4. Module `//!`: role + intuition + why when the shape is a design choice; entrypoints; flow if multi-step?  
5. Body does not re-paste the public blurb?  
6. Unclear code → simplify first?  
7. Short *what* only for regex/hard math when needed?  
8. Settings and chosen numbers: what it controls, what happens at the limit, special values, relations, default; why this value (§ Settings and numbers)?  

### Flag

- Restates the next line  
- Module/API rule with **no why / benefit / alternative** when the rule is a design choice  
- Comment only needed because the code is overcomplicated  
- Body repeats full public blurb  
- Ticket-only with no constraint  
- Missing *why* on truly non-obvious code  
- Hard-to-read comment text  
- Setting doc that tells the incident instead of the behavior, or leaves out what happens at the limit  
- Chosen number with no reason, or provenance only  

Not a flag: short ownership, special values, concurrency, decision rationale, brief *what* on hard algorithms when it helps; tiny modules whose name already is the whole story; well-known protocol numbers (port 443); arbitrary test literals.
