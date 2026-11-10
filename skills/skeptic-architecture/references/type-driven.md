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

## Example E2 — time as data in the state, not a sleep

**Need:** state that changes with time as well as with events — a traffic light, a retry backoff, a lease, a session timeout.

**Bad:** the steps are separated by `sleep`. While it sleeps, the code cannot react: a button pressed during a 30-second green waits until the sleep ends. The state ("green, 12 s in") exists only as *the line the program is on*, so it cannot be printed, checked in a test, or used as a test's starting point. One cycle takes 63 real seconds to test, and each step lasts its sleep plus however long the step before it took.

```rust
loop {
    show(Color::Green);
    sleep(Duration::from_secs(30));
    show(Color::Yellow);
    sleep(Duration::from_secs(3));
    show(Color::Red);
    sleep(Duration::from_secs(30));
}
```

**Good:** the state stores **when it should change**; the current time is passed in; a tick is just another event. Nothing sleeps.

```rust
/// How soon green ends after a pedestrian presses the button, unless it was ending sooner.
const BUTTON_WAIT: Duration = Duration::from_secs(5);

/// Which color is showing and when it should change. The change time is data, so a button
/// can move it and a test can choose it.
pub struct Light {
    pub color: Color,
    pub change_at: Instant,
}

impl Light {
    pub fn start(now: Instant) -> Light {
        Light { color: Color::Green, change_at: now + Color::Green.duration() }
    }

    /// Applies one event at time `now`; returns the new light and what to display, if anything.
    pub fn step(self, event: Event, now: Instant) -> (Light, Option<Command>) {
        match event {
            Event::Tick if now >= self.change_at => {
                let color = self.color.next();
                // Schedule from the planned change time, not from `now`, so a late tick
                // does not push every later change back.
                let change_at = self.change_at + color.duration();
                (Light { color, change_at }, Some(Command::Show(color)))
            }
            Event::Button if self.color == Color::Green => {
                let change_at = self.change_at.min(now + BUTTON_WAIT);
                (Light { change_at, ..self }, None)
            }
            // A tick before the change time, or a button while not green: nothing to do.
            Event::Tick | Event::Button => (self, None),
        }
    }
}

#[test]
fn button_while_green_moves_the_change_to_five_seconds_away() {
    let start = Instant::now();
    let green = Light { color: Color::Green, change_at: start + secs(30) };

    let (light, command) = green.step(Event::Button, start + secs(10));

    assert_eq!(light, Light { color: Color::Green, change_at: start + secs(15) });
    assert_eq!(command, None);
}

#[test]
fn late_tick_keeps_the_schedule() {
    let start = Instant::now();
    let yellow = Light { color: Color::Yellow, change_at: start + secs(18) };

    let (light, _) = yellow.step(Event::Tick, start + secs(20));

    assert_eq!(light, Light { color: Color::Red, change_at: start + secs(48) });
}
```

`Color::next` and `Color::duration` are plain `match`es over the three colors. A struct with a `color` field fits better than an enum of states here because every color carries the same data — the change time (an enum of states pays off when each state holds different data, Example E).

Once the change time is data, the button is one more event: it moves the change time earlier the moment it arrives. The test chooses `now`, so 30 seconds pass in one line. Storing *when to change* rather than counting ticks means a late tick cannot make green longer: whenever the next tick comes, it sees `now >= change_at`. Scheduling the next change from `change_at` rather than from `now` keeps lateness from adding up, which the sleeping loop cannot avoid. The shell that waits for the next tick, a button, or a stop signal is in `skeptic-async/references/select-and-tasks.md` § 2.8.

**Review check:** a long-running loop that separates its steps with `sleep` (sync or async), or counts ticks to measure time? → store the deadline in the state, pass `now` in, and let the shell wait until the deadline or the next event.

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

## Example G — arguments that can be swapped; index, count, and size

**Sources:** TigerStyle ("a function taking two `u64` must use an options struct"; off-by-one errors between index, count, and size); matklad, [Newtype Index Pattern](https://matklad.github.io/2018/06/04/newtype-index-pattern.html).

**Need:** two arguments of the same type that mean different things — accounts to debit and credit, source and target offsets, an index and a count — where a swap compiles and corrupts data.

**Bad:** positional arguments of one primitive type. `transfer(credit, debit, amount)` compiles and moves money the wrong way.

```rust
fn transfer(debit: u64, credit: u64, amount_cents: u64) -> Result<(), TransferError>
```

**Good:** newtypes stop mixing *different kinds* of value (an account id is not an amount); a struct with named fields stops swapping *two values of the same kind*, because the call site names each one:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct AccountId(pub u64);

/// A transfer is built with named fields, so `debit` and `credit` cannot be swapped by position.
pub struct Transfer {
    pub id: TransferId,
    pub debit: AccountId,
    pub credit: AccountId,
    pub amount_cents: u64,
}

let transfer = Transfer { id, debit: buyer, credit: seller, amount_cents: 5_000 };
```

Newtypes alone are not enough here: `debit` and `credit` are both `AccountId`, so `transfer(seller, buyer, …)` would still compile.

**Index, count, and size are different numbers.** An index counts from 0, a count from 1, and a size is a count times a unit. Most off-by-one errors are a casual mix of the three. Name the unit (`size_bytes`, `count`, `index`), give an index into one collection its own type, and let the collection accept only that type:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct SlotIndex(pub usize);

// Allowed by Rust's orphan rule because `SlotIndex` is local. `slots[slot_index]` works, and
// `slots[order_index]` with an `OrderIndex` does not compile. A bare `usize` still indexes the
// slice, so keep raw indexes out of signatures.
impl std::ops::Index<SlotIndex> for [Slot] {
    type Output = Slot;
    fn index(&self, index: SlotIndex) -> &Slot {
        &self[index.0]
    }
}
```

**Say how a division rounds.** `size_bytes / PAGE_SIZE_BYTES` silently rounds down; when a partial page still takes a page, say so:

```rust
/// Pages needed to hold `size_bytes`; a partial page still takes a whole page.
pub fn count_pages(size_bytes: u64) -> u64 {
    size_bytes.div_ceil(PAGE_SIZE_BYTES)
}
```

**Fixed-size integers at the boundary.** In memory, `usize` is the natural index type in Rust. In anything written to disk or sent over the network, use fixed-size types (`u32`, `u64`) so the format is the same on every machine.

**Review check:** two or more arguments of the same type with different meanings at a public or widely used call site? → named fields in a struct, or distinct newtypes when the kinds differ. An index into a specific collection passed as bare `usize` where several collections are in scope? → an index newtype. Division on sizes or counts with no stated rounding? → `div_ceil` or a comment saying round-down is intended.

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
- Finding: “`run_light` shows a color, then `sleep(30 s)`; a button press waits out the sleep, and the state exists only as the paused line. Option: `Light { color, change_at }` with a pure `step(self, event, now)`; the shell waits until `change_at` or the next event.”  
- Finding: “`fn transfer(debit: u64, credit: u64, amount_cents: u64)` — swapping `debit` and `credit` compiles. Option: a `Transfer` struct with named fields and `AccountId` newtypes.”  
- Finding: “`RequestBuilder::build()` fails when `url` was never set, but code always sets it. Option: take `url` in `new`; `build()` stops returning `Result`.”  
- Finding: “new `KafkaRecord` repeats `BrokerRecord` (topic, partition, offset, key, payload). Options: reuse and carry registration beside it; generalize to one record and one payload enum; keep both only because registration does not exist before register.”
