# Exemplars — hazard and ownership comments

**Class:** truths that names, types, and tests do not make cheap. Prefer **why / who must / must not / negative space**.

**Also drawn from (language-agnostic parts of):** [Google C++ Style Guide — Comments](https://google.github.io/styleguide/cppguide.html#Comments). We do **not** copy C++-only rules (header vs `.cc` placement, `//` file banners for includes, etc.). Rust keeps docs on the item (`///` / `//!`).

## Local reasoning

A reader should grasp what a call does **at the call site** (or on the public `///`) without opening every implementation. Types and names do most of that; comments fill the gaps. (“Leave a trace for the reader” / in-place evidence — same goal as Google’s comment rules.)

| Documentation (`//!` / `///`) | Implementation (`//`) |
|------------------------------|------------------------|
| Intent and design of module/type/function | Why this branch, this order, this skip |
| Purpose, contracts, given/expected | Tricky bit, hazard, “do not simplify to X” |

## What each comment kind is for (portable)

| Kind | Put | Job (Google-aligned, Rust form) |
|------|-----|----------------------------------|
| **File / module** | `//!` at top of module | What this unit is for; key invariants; entrypoints / short flow when multi-step |
| **Type / function (public or real contract)** | `///` | Design and intent: purpose, inputs/outputs in plain words or given/expected, failure/no-op, thread/order notes when part of the contract |
| **Inside the body** | `//` | Non-obvious choices, tricky bits, important “do not undo” — the explanation a reader would look for *here* |

Function/module docs describe **what and why** the API exists; body comments justify **surprising how**. Prefer fixing unclear code over a long comment that apologizes for it (“don’t comment bad code — rewrite it” when rewrite is cheap).

## Module purpose + boundary (good pattern)

```rust
//! Cold DuckDB read timing and result artifacts for historian-shaped SQL.
//! Object-storage hydration, gRPC pagination, and production connection pools are out of scope.
//! Local "cold" is DuckDB-cache-cold only; OS page cache may still be warm.
```

Pattern: **purpose + key invariant** at module top (plain language). Optional: where related work lives. No required “owns / does not own” labels. Use denseness without storytelling.

## Multi-step flow (good pattern)

Prefer **context on each step** (inputs, outcomes, important side notes), not bare names only:

```rust
//! ```text
//! connect_to_duckdb() → Connection
//!   → resolve parquet files (non-empty)
//!     → measure_profile → ParquetStats
//!       → write_stats_json (file + human summary)
//! ```
```

Avoid stacking Pure/Shell/Compose inventories that restate the diagram. Pure vs shell is a **code-shape / review** concern, not a documentation template — not on `//!`, and not on function `///` either (“Pure (no I/O)”, “No DuckDB”, “thinking code”). Prefer a real contract or silence.

## Scar at the right level (good pattern)

Backward-compat / serde defaults belong on the **type or field** (and a test), not a vague module “Compat:” line:

```rust
/// `_spread` / `distinct_values` default to 0 so older stats.json still load;
/// generate treats 0 as no per-signal variation / no quantization.
#[serde(default)]
pub(crate) frac_unchanged_spread: f64,
```

## Coherence across modules (when reviewing a crate)

| Check | Fail when |
|-------|-----------|
| Cascade style | One module uses `**Flow**` + bullets, another uses bare arrows, a third lists Pure/Shell |
| Detail level | Sibling private modules: one has an essay, next has nothing |
| Names match code | Diagram says old fn name after rename |
| Placement | History/compat only on module; fields that carry `#[serde(default)]` have no note |
| No template theater | Forced owns/not-owns or Pure/Shell on every `//!` / `///` |

## Identity / pairing hazard (good pattern)

```rust
// Workload key joins signals with a unit separator so one signal named "a,b"
// does not collide with two signals "a" and "b". Do not switch to comma join.
fn workload_key(...) -> String { ... }
```

## Security hazard (good pattern)

```rust
// CREATE SECRET so keys are redacted in DuckDB errors/settings; map_err so our
// built SQL (which still contains literals) is not attached to the anyhow chain.
conn.execute_batch(&secret)
    .map_err(|_| anyhow::anyhow!("failed to configure object storage credentials"))?;
```

## Scar-style (good pattern)

When the same mistake bit you twice, one local line:

```rust
// Stale writer must not overwrite newer state — generation check is the contract.
```

Public Ghostty/Mitchell writing emphasizes **scar logs** and mechanisms over slogans: record the failure mode next to the code or in a short scars list that feeds hard-rules — not a novel in every function.

## Netflix-style denseness (pattern, not a paste)

High-density industrial code comments tend to pack **caller obligations + negative space + policy** into few lines, not “// increment i”. When reviewing, prefer:

- one block that changes how you call or change the code  
- over many lines that narrate the algorithm  

If you have an approved internal sample, add it here with path.

## Bad

```rust
// Loop over files
for f in files {
    // Get the name
    let name = f.name();
```

Restates the next line. Delete.

```rust
// TEI-34468: skip cache
```

Weak: ticket without the real constraint. Prefer the **why** first; add a ticket only if it is a lasting scar (e.g. critical prod bug):

```rust
// OS page cache not cleared on local cold — use remote for true cold.
// Critical: TEI-34468 — async A/B misread warm local as engine win.
```
