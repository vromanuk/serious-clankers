# Design headers (purpose + given/expected)

On non-trivial functions: **purpose** (what it does **and why this shape** when that is a design choice), then **given / expected**.  
**No type-signature line** — types live in the type system.

Purpose is not a synonym of the function name. If the reader might ask “why not the obvious alternative?”, answer in one phrase (see `comments.md` § why bar).

## Good (Rust)

```rust
/// Half-open read window from data range and optional span/end.
///
/// End-exclusive so adjacent windows tile without double-counting the boundary sample.
///
/// given: range min=0 max=200_000_000, no window, no explicit end
/// expected: start=0, end=200_000_001
///
/// given: window_ms=900_000, end=1_000_000_000
/// expected: start=999_100_000, end=1_000_000_000 (data max ignored)
fn window(...) -> Window { ... }
```

```rust
/// Scale each signal class to target packs; hold per-series sample rate.
///
/// Holds rates so cardinality stays realistic when pack count changes (avoids
/// “tiny packs, huge series” that mis-trains query plans).
///
/// given: measured 489 packs, 1000 signals, total_samples = 489*1000*100, target 250 packs
/// expected: samples_per_series = 100, packs = 250
fn scale_plan(...) -> Vec<ClassPlan> { ... }
```

## Bad

```rust
/// window: (i64, i64) × Option<i64> × Option<i64> -> Window
/// Takes range and returns Window.
fn window(...) -> Window { ... }
```

Why bad: type line + restates the name; no **why**, no examples of the contract.

```rust
/// Returns a half-open window.
fn window(...) -> Window { ... }
```

Why weak: pure restatement of the name; no contract examples and no reason for half-open.

## When to skip

Tiny leaves already named by the caller (`esc`, `basename`, one-line getters).
