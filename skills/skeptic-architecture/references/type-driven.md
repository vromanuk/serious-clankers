# Type-driven contracts

**Sources:** [Type-Driven API Design in Rust](https://willcrichton.net/rust-api-type-patterns/introduction.html) (Will Crichton); [Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/); [The Typestate Pattern in Rust](https://cliffle.com/blog/rust-typestate/) (Cliff Biffle); [How To Use The Typestate Pattern In Rust](https://zerotomastery.io/blog/rust-typestate-patterns/) (Jayson Lennon); [Enums as State Machines](https://learn.sunnstech.com/tutorials/rust/enums-as-state-machines/) and [Builder Pattern and Typestate](https://learn.sunnstech.com/tutorials/rust/builder-pattern-and-typestate/) (Sunns Technologies); Rust API Guidelines.

Use when reviewing APIs, return types, or “caller must remember to check X.”

## Main idea (short)

Map the real concept 1:1 into types (**cardinality**). When several pieces must stay in sync, make that **impossible to break** at compile time (or at one construction site), not by comment or a second helper.

| Pattern | Plain meaning | When to use |
|---------|---------------|-------------|
| **Enum over string / bool soup** | Finite alternatives are variants | Modes, query kinds, auth kinds |
| **Checked success type** | Value only exists after a property holds; private construct | “Must have files”, “profile passed checks”, “creds decided” |
| **RAII / lock guard** | Type that both proves a state and controls access/cleanup | Mutex guards, open sessions (std already does this) |
| **State in the type** (typestate) | Different methods per step of a protocol; a step consumes the old state | Caller-driven multi-step flows where wrong order is a bug (Example D); not loops driven by incoming events |
| **Enum of states** | Each variant holds only that state’s data; a transition takes `self` and returns the next state | State that changes at run time — events, IO, correlated flags (Example E) |
| **Required inputs in the constructor** | A value with no default is a `new` argument, not an `Option` checked in `build()` | Builders and configs with required fields (Example F) |
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
- Callers outside the module cannot build a later state directly: state types keep private fields and have no public constructor. Otherwise `ParsedText(words)` skips `parse()`, and the order is back to a convention.
- A transition allowed from every state (fault, abort, cancel) is written once in `impl<S>`, next to one private `transition` that moves the shared fields. It is not copied into each state’s `impl`.

```rust
impl<S> Response<S> {
    fn transition<N>(self, next: N) -> Response<N> {
        Response { inner: self.inner, state: next }
    }
    pub fn abort(self) -> Response<Aborted> { self.transition(Aborted) } // from any state
}
```

**Fits:** the caller drives the steps and always knows which step it is at — builders, connections, protocol encoders (serde’s `Serializer` → `SerializeStruct` → `end`).

**Does not fit:** a machine where incoming events decide the next state (a Kafka consumer, an elevator, a Raft node). The code holding it cannot know the state at compile time, so use an enum of states (Example E) and check invariants in tests. Also not worth it for a two-step script that never stops between steps.

**Review check:** is an order the caller controls checked at runtime (a “wrong stage” error, or an `Option` filled in one step and unwrapped in a later one)? → offer typestate. Is typestate forced onto a loop whose state changes with each incoming event? → offer an enum of states.

---

## Example E — state that changes at run time (enum of states)

**Source:** [Enums as State Machines](https://learn.sunnstech.com/tutorials/rust/enums-as-state-machines/) (Sunns Technologies).

**Need:** a connection, job, or session moves between states as events arrive. The code holding it learns the state only at run time, so typestate cannot help.

**Bad:** one field per condition. Four fields allow 16 combinations; a handful are legal. Nothing stops `is_connected && is_connecting`, or a `session_id` while disconnected, and every function re-checks the combinations.

```rust
struct Connection {
    is_connected: bool,
    is_connecting: bool,
    session_id: Option<String>, // only meaningful when connected
    retry_count: u32,           // only meaningful when connecting
}
```

**Signal:** the second correlated flag — two bools, or a bool plus an `Option` that is valid only when the bool is set. Convert before a third one arrives. Example B is the same fix for exclusive settings; this is the fix for state over time.

**Good:** one variant per state, holding only that state’s data. A transition takes `self` and returns the next state, so the old one cannot be used again.

```rust
enum Connection {
    Disconnected,
    Connecting { retry_count: u32 },
    Connected { session_id: String },
    Failed { error: String },
}

impl Connection {
    fn on_session(self, session_id: String) -> Result<Connection, (Connection, NotConnecting)> {
        match self {
            Connection::Connecting { .. } => Ok(Connection::Connected { session_id }),
            state @ (Connection::Disconnected
            | Connection::Connected { .. }
            | Connection::Failed { .. }) => Err((state, NotConnecting)),
        }
    }
}
```

A transition that does not apply in the current state gets one of three answers, chosen by what it means:

| The event in this state is | Answer |
|---|---|
| Expected and recoverable | `Result<Next, (Self, Error)>` — the state comes back with the reason, so the caller can retry or branch |
| Irrelevant | An arm that names those states and returns `self` unchanged |
| Impossible unless the code is wrong | Panic with the invariant (`unreachable!("…")`) |

A catch-all arm (`other => other`, `_ => Connection::Disconnected`) is none of these: it silently keeps or invents a state, and a new variant compiles without anyone deciding its transitions. Listing the states keeps the `match` exhaustive; clippy’s `wildcard_enum_match_arm` checks this mechanically.

Transitions are data in, data out — a pure `step(state, event)` that sequence or property tests cover without IO (skeptic-testability).

**Review check:** two correlated flags, or an `Option` valid only under a flag? → enum of states. A transition `match` with a catch-all arm? → name the states and pick one of the three answers.

---

## Example F — required fields (builders)

**Source:** [Builder Pattern and Typestate](https://learn.sunnstech.com/tutorials/rust/builder-pattern-and-typestate/) (Sunns Technologies).

**Need:** a type with a few values that have no sensible default and several that do.

**Bad:** the required value is an `Option` that `build()` checks. Whether it was set is under the caller’s control, yet forgetting it compiles and fails at run time — the same smell as a stage field in Example D.

```rust
#[derive(Default)]
struct RequestBuilder { url: Option<Url>, method: Option<Method> }

impl RequestBuilder {
    fn build(self) -> Result<Request> {
        let url = self.url.ok_or_else(|| anyhow!("url is required"))?;
        Ok(Request { url, method: self.method.unwrap_or(Method::GET) })
    }
}
```

**Good:** required values are `new` arguments; optional ones have defaults and override methods; `build()` cannot fail on a missing field.

```rust
impl RequestBuilder {
    fn new(url: Url) -> Self { Self { url, method: Method::GET } }
    fn method(mut self, method: Method) -> Self { self.method = method; self }
    fn build(self) -> Request { Request { url: self.url, method: self.method } }
}
```

In order of preference:

1. **Required → constructor argument.** When two required arguments share a type and could be swapped, give them newtypes (`Topic`, `GroupId`).
2. **Optional → default plus override method**, or an options struct with `Default` (skeptic-solid `package-principles.md`).
3. **Typestate builder** (one marker per required field, `build` only on the all-set state) → only for a public API with many required fields, where even newtype arguments read poorly. It costs a type per field and harder rustdoc.
4. **`Option` + `build() -> Result`** → right when the values arrive at run time from config, files, or requests. That is parsing at the edge (Example C): done once, with an error that names the missing key.

**Review check:** a required field stored as an `Option` in a builder that code (not config) fills? → move it to `new`. A typestate builder for an internal type with one or two required fields? → constructor arguments.

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
- A catch-all arm in a transition `match` that keeps or defaults the state without saying why.  
- Typestate state types that callers outside the module can build directly.  
- A typestate builder where constructor arguments would do.  
- Non-empty success type **and** a second free `require_*` that still takes bare `Vec` (undermines the type).  
- Copy-pasted error strings instead of one constructor.  
- A second struct that copies an existing one field-for-field under new names. Generalize or reuse; do not keep the copy without naming the fact that would be a lie if they were one type.

---

## Review one-liners

- `type-driven: ok` — e.g. “`NonEmptyParquetFiles` forces empty fail at list/resolve; generate reuse stays on `Vec`.”  
- Finding: “empty check only in comments / second helper; success type still allows empty.”  
- Finding: “parallel `Option`s for exclusive modes; encode as enum at decide site.”  
- Finding: “`header()` checks `self.stage` at runtime, but the caller controls the order. Option: typestate — `status_line(self) -> Response<Headers>`, `header` only on `Response<Headers>`, `body(self)` consumes.”  
- Finding: “`is_connected`, `is_connecting`, and `session_id: Option` must agree, and nothing enforces it. Option: enum of states with `session_id` inside `Connected`.”  
- Finding: “`on_session` ends in `other => other`, so a session event while `Failed` is silently dropped. Options: return `Err((state, NotConnecting))`; name the states where it is a no-op; `unreachable!` if it cannot happen.”  
- Finding: “`RequestBuilder::build()` fails when `url` was never set, but code always sets it. Option: take `url` in `new`; `build()` stops returning `Result`.”  
- Finding: “new `KafkaRecord` repeats `BrokerRecord` (topic, partition, offset, key, payload). Options: reuse and carry registration beside it; generalize to one record and one payload enum; keep both only because registration does not exist before register.”
