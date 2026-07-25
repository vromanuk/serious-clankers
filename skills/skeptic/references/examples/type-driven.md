# Type-driven contracts

**Sources:** [Type-Driven API Design in Rust](https://willcrichton.net/rust-api-type-patterns/introduction.html) (Will Crichton); [Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/); Rust API Guidelines.

Use when reviewing APIs, return types, or “caller must remember to check X.”

## Main idea (short)

Map the real concept 1:1 into types (**cardinality**). When several pieces must stay in sync, make that **impossible to break** at compile time (or at one construction site), not by comment or a second helper.

| Pattern | Plain meaning | When to use |
|---------|---------------|-------------|
| **Enum over string / bool soup** | Finite alternatives are variants | Modes, query kinds, auth kinds |
| **Checked success type** | Value only exists after a property holds; private construct | “Must have files”, “profile passed checks”, “creds decided” |
| **RAII / lock guard** | Type that both proves a state and controls access/cleanup | Mutex guards, open sessions (std already does this) |
| **State in the type** | Different methods per step of a protocol | Multi-step flows where wrong order is a bug |
| **Parse at the edge** | `&str` / CLI → domain type once | Don’t re-check the same rule in three places |

Do **not** reach for heavy state-in-type machinery by default. Prefer the **smallest** type that removes the real forgettable bug.

---

## Example A — non-empty product listing

**Need:** product reads must have ≥1 parquet file; inventory/reuse may see zero.

**Bad:** success is `Vec` + each call site `if empty { bail! }` (strings diverge; one site forgets).

```rust
// Bad: empty is still a successful Vec
fn resolve_files(source: &Source) -> Result<Vec<ParquetLocation>> { /* ... */ }

// Call site can forget:
let files = list_parquet()?;
// measure(files)  // silent zero-file run
```

**Good:** construction is the only place empty fails; product APIs return that type.

```rust
/// Non-empty listing. Only `try_from_files` builds it.
pub(crate) struct NonEmptyParquetFiles {
    files: Vec<ParquetLocation>, // private
}

impl NonEmptyParquetFiles {
    fn try_from_files(files: Vec<ParquetLocation>, label: &str) -> Result<Self> {
        if files.is_empty() {
            anyhow::bail!("no .parquet files found for {label} — check the path or bucket/prefix");
        }
        Ok(Self { files })
    }
}

// Product paths — empty cannot be ignored:
fn resolve_files(source: &Source) -> Result<NonEmptyParquetFiles> { /* ... */ }
fn list_parquet(&self) -> Result<NonEmptyParquetFiles> { /* same try_from_files */ }

// Inventory path — different job, different type:
fn find_parquet(path: &str) -> Result<Vec<ParquetLocation>> { /* empty Ok */ }
```

**Review check:** if two call sites share “must have files,” do they share **one success type**, or two bails?

---

## Example B — mutually exclusive credentials (enum, not Option soup)

**Need:** profile **xor** access+secret keys.

**Bad:** parallel options; every consumer re-decides.

```rust
// Bad: caller can pass both or neither
profile: Option<String>,
access_key: Option<SecretString>,
secret_key: Option<SecretString>,
```

**Good:** decide once into an enum; resolve later.

```rust
enum ObjectStorageCredentials {
    Profile(String),
    AccessKeys { access: SecretString, secret: SecretString, session: Option<SecretString> },
}

impl ObjectStorageCredentials {
    fn from_cli(/* flags */) -> Result<Self> { /* reject both / neither once */ }
    fn resolve(self) -> Result<ResolvedCredentials> { /* load profile or use keys */ }
}
```

**Review check:** can a later function receive an impossible combination without a type error?

---

## Example C — parse at the edge (window / seed / color)

**Need:** CLI string becomes domain data once.

```rust
// Bad: three places parse "15m"
window: Option<String>,

// Good:
struct WindowMs(u64);
fn parse_window_ms(s: &str) -> Result<WindowMs> { /* once at CLI boundary */ }
// inner code only sees WindowMs / EpochMs
```

Same idea as Crichton’s `PrimaryColor` vs `&str`: after parse, the function cannot fail for “unknown color.”

---

## What *not* to do

- Newtype every `Vec` when empty is a valid **inventory** outcome.  
- Typestate for a two-step script that never returns mid-protocol.  
- Non-empty success type **and** a second free `require_*` that still takes bare `Vec` (undermines the type).  
- Copy-pasted error strings instead of one constructor.

---

## Review one-liners

- `type-driven: ok` — e.g. “`NonEmptyParquetFiles` forces empty fail at list/resolve; generate reuse stays on `Vec`.”  
- Finding: “empty check only in comments / second helper; success type still allows empty.”  
- Finding: “parallel `Option`s for exclusive modes; encode as enum at decide site.”
