# Design headers (purpose + given/expected)

On non-trivial functions: purpose (what/why), then **given / expected**.  
**No type-signature line** — types live in the type system.
## Good (Rust)

```rust
/// Half-open read window from data range and optional span/end.
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

Why bad: type line + restates the name; no examples of the contract.

## When to skip

Tiny leaves already named by the caller (`esc`, `basename`, one-line getters).
