# Hazard and ownership comments

Comments should add what names and types don’t already make clear: **why**, who must/must not, what not to break.

**Google comment ideas (any language) + review list:** load `google-comments.md`  
([source](https://google.github.io/styleguide/cppguide.html#Comments)).  
Rust: put docs on the item (`///` / `//!`).

**Review in one line:** needed? clear English? **why** (decision), not **what**? if the code is unclear, simplify it — comments are for what the code can’t say (reasoning; short *what* is OK for regex / hard algorithms).

## Easy to understand at the use site

A reader should get what a call does **where it’s used** (or on the public `///`) without opening every body. Names and types do most of that; comments fill gaps.

| On the public item (`//!` / `///`) | Inside the body (`//`) |
|------------------------------------|------------------------|
| Purpose, how to use, contracts | Why this branch, order, or skip |
| Given/expected when useful | Hazard, “do not change to X” |

| Kind | Put | Job |
|------|-----|-----|
| **Module** | `//!` | What this unit is for; key rules; short flow when multi-step |
| **Type / function** | `///` | Purpose, contracts, failure, order notes when part of the API |
| **Body** | `//` | Non-obvious choices next to the hard line |

Prefer fixing unclear code over a long comment that apologizes for it.

## Module purpose + boundary

```rust
//! Cold DuckDB read timing and result artifacts for historian-shaped SQL.
//! Object-storage hydration, gRPC pagination, and production connection pools are out of scope.
//! Local "cold" is DuckDB-cache-cold only; OS page cache may still be warm.
```

**purpose + key invariant** at module top. Optional: where related work lives. No forced “owns / does not own” template. Dense when useful; no storytelling essay.

## Multi-step flow

Prefer **context on each step** (inputs, outcomes, side notes), not bare names only:

```rust
//! ```text
//! connect_to_duckdb() → Connection
//!   → resolve parquet files (non-empty)
//!     → measure_profile → ParquetStats
//!       → write_stats_json (file + human summary)
//! ```
```

Do not stamp labels like “pure” / “shell” on every doc. Say the real rule in ordinary words, or say nothing.

## Scar at the right level

Compat / serde defaults belong on the **type or field** (and a test), not a vague module “Compat:” line:

```rust
/// `_spread` / `distinct_values` default to 0 so older stats.json still load;
/// generate treats 0 as no per-signal variation / no quantization.
#[serde(default)]
pub(crate) frac_unchanged_spread: f64,
```

## Coherence across modules

| Check | Fail when |
|-------|-----------|
| Cascade style | Modules mix unrelated flow styles with no pattern |
| Detail level | Sibling private modules: one essay, next empty |
| Names match code | Diagram still uses old function names |
| Placement | History only on module; `#[serde(default)]` fields have no note |
| No empty templates | Forced “owns / does not own” or pure/shell labels on every doc |

## Identity / pairing hazard

```rust
// Workload key joins signals with a unit separator so one signal named "a,b"
// does not collide with two signals "a" and "b". Do not switch to comma join.
fn workload_key(...) -> String { ... }
```

## Security hazard

```rust
// CREATE SECRET so keys are redacted in DuckDB errors/settings; map_err so our
// built SQL (which still contains literals) is not attached to the error chain.
conn.execute_batch(&secret)
    .map_err(|_| anyhow::anyhow!("failed to configure object storage credentials"))?;
```

## Scar line (when the same miss happened twice)

```rust
// Stale writer must not overwrite newer state — generation check is the contract.
```

One local line (or a hard-rule). Not a novel in every function.

## Dense comments (when they help)

Prefer a short block that changes how you **call or change** the code (caller obligations, negative space, policy) over many lines that narrate the algorithm (“// increment i”).

## Bad — states the obvious

Same rule as Google: do not describe what the next line already says. Prefer **why**, or make the code self-describing (see `google-comments.md`).

```rust
// Loop over files
for f in files {
    // Get the name
    let name = f.name();
```

Restates the next line. Delete (or rename / extract until no comment is needed).

```rust
// TICKET-123: skip cache
```

Ticket without the constraint. Prefer **why** first; ticket only if it is a lasting scar:

```rust
// OS page cache not cleared on local cold — use remote for true cold.
// Scar: local warm runs were misread as engine wins (TICKET-123).
```
