# Exemplars — Rust API and type shape

**Sources:** [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/); Jon Gjengset (*Rust for Rustaceans*) themes; [Type-Driven API Design in Rust](https://willcrichton.net/rust-api-type-patterns/introduction.html) (deeper patterns → **`type-driven.md`**).

## Prefer types that make bad states hard

```rust
// Good: run identity is data, not stringly starts_with("baseline")
enum RunLabel { Baseline, Candidate }

// Good: window parsed once at the edge
struct WindowSpec { label: String, ms: i64 }
```

```rust
// Bad: re-parse and hope
window: Option<String>,  // "15m" re-parsed in three places
label: String,           // "baseline_*" convention is the only identity
```

## C-CUSTOM-TYPE / misuse resistance

- Newtypes when two `u64`s mean different things (`Packs`, ids).  
- Enums for real alternatives, not magic strings at internal boundaries.  
- Parse at the edge; inner code uses domain types.  
- **Witness / non-empty success types** when “empty is an error” must not be forgettable — full worked examples in **`type-driven.md`**.

## Ownership (Gjengset-style)

- Prefer `&str` / `&[T]` over `&String` / `&Vec<T>`.  
- Do not `clone` only to end a borrow fight — fix ownership.  
- Document who may touch shared state; keep mutex spans short if you must use them.

## Errors

- Binary/tool: `anyhow` + context is fine.  
- Library: concrete error types.  
- No `unwrap` in library paths; `expect` only with invariant text.

## Not for this file

Full clippy pedantic lists — run `cargo clippy` instead (LLM is not a linter).
