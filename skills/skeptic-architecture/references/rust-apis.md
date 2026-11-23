# Rust API and type shape

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

- Newtypes when two `u64`s mean different things (`DeviceCount`, ids).  
- Newtype with a private field and `parse` when a `String` or number must satisfy a rule; a type alias only names a type and never stops a mix-up. Which to pick, and when a plain primitive is right: **`type-driven.md` § Example H**.  
- No `pub type X = String` (or `= u64`) at a crate boundary: callers then depend on the primitive, and moving to a newtype later breaks them.  
- Enums for real alternatives, not magic strings at internal boundaries.  
- Parse at the edge; inner code uses domain types.  
- **Witness / non-empty success types** when “empty is an error” must not be forgettable — full worked examples in **`type-driven.md`**.

## Ownership (Gjengset-style)

- Prefer `&str` / `&[T]` over `&String` / `&Vec<T>`.  
- Do not `clone` only to end a borrow fight — fix ownership.  
- Document who may touch shared state; keep mutex spans short if you must use them.

**The signature shows what the function needs** (Apollo *Rust Best Practices*, chapters 1 and 3). A caller should
see the cost of a call without reading the body:

```rust
// Worse: borrows, then clones inside; the caller cannot see the copy, and cannot avoid it by moving
pub fn register(&mut self, device: &Device) { self.devices.push(device.clone()); }

// Better: takes ownership; a caller that is done with the value moves it, one that is not clones it visibly
pub fn register(&mut self, device: Device) { self.devices.push(device); }
```

- Take `T` when the function stores or consumes the value; take `&T` when it only reads.  
- Small `Copy` values (a few machine words, no heap data) pass by value; large values (hundreds of bytes) pass by
  reference or in a `Box`.  
- `Cow<'_, str>` when a function usually returns its input unchanged and only sometimes builds a new value.  
- Never make a type both `Copy` and `Iterator`: a silent copy advances separately from the original, so code that
  looks like it continues one iteration restarts it. Implement `IntoIterator` and return a separate iterator type.  

The mechanical cases have clippy lints (`ptr_arg`, `needless_pass_by_value`, `trivially_copy_pass_by_ref`,
`large_types_passed_by_value`, `copy_iterator`; `skeptic-conventions/references/lint-baseline.md`). The judgment
case — a borrow that is cloned inside — has none, so check it here.

## Errors

- Binary/tool: `anyhow` + context is fine.  
- Library: concrete error types.  
- No `unwrap` in library paths; `expect` only with invariant text.

## Not for this file

Full clippy pedantic lists — run `cargo clippy` instead (LLM is not a linter).
