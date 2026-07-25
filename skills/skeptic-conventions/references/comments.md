# Hazard and ownership comments

Comments carry truths that names, types, and tests do not make cheap: **why / who must / must not / negative space**.

Useful portable ideas: [Google C++ Style Guide — Comments](https://google.github.io/styleguide/cppguide.html#Comments) (intent, not narration). Rust: put docs on the item (`///` / `//!`).

## Local reasoning

A reader should grasp what a call does **at the call site** (or on the public `///`) without opening every implementation. Types and names do most of that; comments fill the gaps.

| Documentation (`//!` / `///`) | Implementation (`//`) |
|------------------------------|------------------------|
| Intent and design of module/type/function | Why this branch, this order, this skip |
| Purpose, contracts, given/expected | Tricky bit, hazard, “do not simplify to X” |

| Kind | Put | Job |
|------|-----|-----|
| **Module** | `//!` | What this unit is for; key invariants; entrypoints / short flow when multi-step |
| **Type / function** | `///` | Purpose, contracts, failure/no-op, order notes when part of the API |
| **Body** | `//` | Non-obvious choices and hazards next to the line that can go wrong |

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

Do not stamp Pure/Shell/Compose inventories on `//!` or `///`. Pure vs shell is a review lens — state real contracts in ordinary words, or say nothing.

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
| No template theater | Forced owns/not-owns or Pure/Shell on every doc comment |

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

## Bad

```rust
// Loop over files
for f in files {
    // Get the name
    let name = f.name();
```

Restates the next line. Delete.

```rust
// TICKET-123: skip cache
```

Ticket without the constraint. Prefer **why** first; ticket only if it is a lasting scar:

```rust
// OS page cache not cleared on local cold — use remote for true cold.
// Scar: local warm runs were misread as engine wins (TICKET-123).
```
