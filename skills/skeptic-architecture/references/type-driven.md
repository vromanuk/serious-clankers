# Type-driven contracts

**Sources:** [Type-Driven API Design in Rust](https://willcrichton.net/rust-api-type-patterns/introduction.html) (Will Crichton); [Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/); [The Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/) (Cliff Biffle); Rust API Guidelines.

Use when reviewing APIs, return types, or “caller must remember to check X.”

## Main idea (short)

Map the real concept 1:1 into types (**cardinality**). When several pieces must stay in sync, make that **impossible to break** at compile time (or at one construction site), not by comment or a second helper.

| Pattern | Plain meaning | When to use |
|---------|---------------|-------------|
| **Enum over string / bool soup** | Finite alternatives are variants | Modes, query kinds, auth kinds |
| **Checked success type** | Value only exists after a property holds; private construct | “Must have files”, “profile passed checks”, “creds decided” |
| **RAII / lock guard** | Type that both proves a state and controls access/cleanup | Mutex guards, open sessions (std already does this) |
| **State in the type** (typestate) | Different methods per step of a protocol; a step consumes the old state | Caller-driven multi-step flows where wrong order is a bug (Example D); not loops driven by incoming events |
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

## Example D — order of operations in the type (typestate)

**Source:** [The Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/) (Cliff Biffle).

**Need:** an order rule the caller controls — “send messages only after login, and never after logout”, “no reads after close”, “status line, then headers, then body”.

**Bad:** one type, a stage field, and a runtime check in each method. Wrong order compiles and fails in production — or the check is simply missing.

```rust
struct Response { stage: Stage, /* ... */ }

impl Response {
    fn header(&mut self, key: &str, value: &str) -> Result<()> {
        if self.stage != Stage::Headers { bail!("header before status line or after body") }
        /* ... */
    }
}
```

**Good:** a method exists only on the state that allows it, and a step takes `self` by value and returns the next state, so the old state cannot be used again.

```rust
struct Response<S> { inner: Box<Inner>, state: S }
struct Start;
struct Headers { code: u16 }   // data that exists only in this state

impl Response<Start> {
    fn status_line(self, code: u16) -> Response<Headers> { /* ... */ }
}

impl Response<Headers> {
    fn header(&mut self, key: &str, value: &str) { /* ... */ }  // no state change → &mut self
    fn body(self, text: &str) { /* ... */ }                      // consumes: nothing after the body
}

impl<S> Response<S> {
    fn bytes_so_far(&self) -> usize { /* valid in every state */ }
}
```

Three parts make it work: operations only on the states that allow them; states written as types; steps that consume the old state. Rust’s move is what makes the third one hold — after `body(self)` the response is gone, while a C++ move leaves a usable moved-from object behind.

Details worth checking:

- An operation that does not change the state takes `&mut self`, so a loop does not need `r = r.header(..)`.
- Data that exists only in some states lives in that state’s type (`Headers { code }`), not in an `Option` that every state must handle.
- The value moved at each step stays cheap (shared data behind a `Box`). Seal the state trait when outsiders must not invent states.
- One type per state reads simplest. A state type parameter (`Response<S>`) suits methods shared by all or several states (`impl<S>`, or `impl<S: Sending>`).

**Fits:** the caller drives the steps and always knows which step it is at — builders, connections, protocol encoders (serde’s `Serializer` → `SerializeStruct` → `end`).

**Does not fit:** a machine where incoming events decide the next state (a Kafka consumer, an elevator, a Raft node). The code holding it cannot know the state at compile time, so use an enum of states with one handler per state, and check invariants in tests. Also not worth it for a two-step script that never stops between steps.

**Review check:** is an order the caller controls checked at runtime (a “wrong stage” error, or an `Option` filled in one step and unwrapped in a later one)? → offer typestate. Is typestate forced onto a loop whose state changes with each incoming event? → offer an enum of states.

---

## New struct vs one that already exists

**Need:** a new struct only when no existing type can be reused or generalized.

Before adding a struct, look in this crate’s types and in the same component. Same fields under new names is a second copy, even when one is `pub` and one is `pub(crate)`, and even when a comment says the new struct is “not” some third type.

**Smell:** `KafkaRecord` (what the processor receives) repeats `BrokerRecord` (a broker record before it is registered). Topic, partition, offset, and key are the same facts. `record_offset` / `offset` and `body` / `payload` are renames. `Body` (`Payload` / `Filtered`) restates `RecordPayload` (`Payload` / `FilteredPayload`). The same field list also sits on `PollingEvent::Record`. The only field `BrokerRecord` does not have is `record_registration`.

```text
BrokerRecord                         KafkaRecord
  topic: String                        topic: String
  partition: i32                       partition: i32
  offset: i64                          record_offset: i64
  key: Vec<u8>                         key: Vec<u8>
  payload: RecordPayload               body: Body
                                       record_registration: RecordRegistration
```

Highlight this. Do not decide it off-stage. The finding offers:

- **Reuse.** Keep `BrokerRecord` (or the `PollingEvent` record). Carry `record_registration` beside it after register, instead of copying topic, partition, offset, key, and body.  
- **Generalize.** One record type and one payload enum. The processor value is that record plus registration. `Filtered` and `FilteredPayload` are one variant.  
- **Keep both** only if a named fact would be a lie on the earlier type. Registration is not a broker record yet — that can be the reason. Renaming fields is not. If this option wins, the finding says the fact in one sentence. A comment that only says “not a row-mapping record” does not.

**Not a flag:** `Registered { record: BrokerRecord, registration: RecordRegistration }` — that reuses the record. Two structs that share one or two incidental fields but not the concept. A newtype that exists only to enforce an invariant the other type does not have.

---

## What *not* to do

- Newtype every `Vec` when empty is a valid **inventory** outcome.  
- Typestate for a two-step script that never returns mid-protocol.  
- Typestate for a machine driven by incoming events — the state is only known at run time; use an enum of states.  
- Non-empty success type **and** a second free `require_*` that still takes bare `Vec` (undermines the type).  
- Copy-pasted error strings instead of one constructor.  
- A second struct that copies an existing one field-for-field under new names. Generalize or reuse; do not keep the copy without naming the fact that would be a lie if they were one type.

---

## Review one-liners

- `type-driven: ok` — e.g. “`NonEmptyParquetFiles` forces empty fail at list/resolve; generate reuse stays on `Vec`.”  
- Finding: “empty check only in comments / second helper; success type still allows empty.”  
- Finding: “parallel `Option`s for exclusive modes; encode as enum at decide site.”  
- Finding: “`header()` checks `self.stage` at runtime, but the caller controls the order. Option: typestate — `status_line(self) -> Response<Headers>`, `header` only on `Response<Headers>`, `body(self)` consumes.”  
- Finding: “new `KafkaRecord` repeats `BrokerRecord` (topic, partition, offset, key, payload). Options: reuse and carry registration beside it; generalize to one record and one payload enum; keep both only because registration does not exist before register.”
