# Complexity

Reference for reviewing whether a change makes code harder to understand or change. Source of the ideas and red flag names: John Ousterhout, *A Philosophy of Software Design* (2nd ed.).

How to use: walk § 3 Red flags on each new or changed unit (function, type, module, error path). A red flag is a reason to look for a simpler design, not an automatic reject. Every rule has limits; each entry lists them.

---

## 1. The model

**Complexity** is anything in the structure of the code that makes it hard to understand or change.

| Symptom | Meaning |
|---------|---------|
| **Change amplification** | One simple change needs edits in many places |
| **Cognitive load** | A lot must be known to make a change safely |
| **Unknown unknowns** | It is not clear what must change, or what must be known — the worst of the three, because the miss shows up as a bug later |

| Cause | Meaning |
|-------|---------|
| **Dependency** | A piece of code cannot be understood or changed on its own |
| **Obscurity** | Important information is not obvious |

Dependencies cause change amplification and cognitive load; obscurity causes unknown unknowns. Dependencies cannot all be removed; keep them few and obvious — prefer a link the compiler checks (a shared name or type) over a hidden one (two places that must agree by convention).

Rules that follow:

- **Complexity builds up in small steps.** A small added dependency is not acceptable because it is small.
- **The reader decides.** If a reviewer finds code hard to follow, it is complex, whatever the author thinks.
- **Weight by use.** Complexity on paths people change often costs far more than in code nobody touches.
- **Line count is a poor measure.** More lines can be simpler when they lower what a reader must hold in mind.

---

## 2. Design principles

1. Complexity builds up in small steps; small things matter.
2. Working code is not enough; the design must stay good too.
3. Improve the design a little with every change.
4. Modules should be deep: a simple interface over a lot of functionality.
5. Make the most common use as simple as possible.
6. A simple interface matters more than a simple implementation.
7. General-purpose modules are deeper.
8. Keep general-purpose and special-purpose code apart.
9. Each layer should offer a different abstraction from the layers next to it.
10. Pull complexity down into the module instead of pushing it to callers.
11. Define errors and special cases out of existence.
12. Design it twice: compare really different options.
13. Comments describe what the code cannot say.
14. Design for ease of reading, not ease of writing.
15. Grow software one abstraction at a time, not one feature at a time.

"Module" means any unit with an interface and an implementation: function, method, type, module, crate, service. The **interface** is everything a caller must know to use it: the signature, and also behavior, ordering rules, side effects, ownership and failure. The unwritten part is usually the larger one.

---

## 3. Red flags

Owner says which skeptic stage reports it. This stage also reports the design problem behind "Hard to Pick Name" and "Hard to Describe"; the wording fix stays with naming and comments.

| Red flag | Meaning | Owner |
|----------|---------|-------|
| Shallow Module | Interface not much simpler than the implementation | complexity |
| Information Leakage | One design decision is known in several places | complexity |
| Temporal Decomposition | Code is split by when it runs, not by what it knows | complexity |
| Overexposure | Using a common feature requires learning rare ones | complexity |
| Pass-Through Method | Does little but forward to a similar method | complexity |
| Repetition | Nontrivial code repeated | complexity |
| Special-General Mixture | General mechanism contains one use's special code | complexity |
| Conjoined Methods | One can't be understood without reading the other | complexity |
| Comment Repeats Code | Comment says only what the code already says | comments |
| Implementation Documentation Contaminates Interface | Public docs describe internals callers don't need | comments |
| Vague Name | Name too broad to say what it is | naming |
| Hard to Pick Name | No precise, simple name fits | complexity (design) + naming (wording) |
| Hard to Describe | A complete doc comment has to be long | complexity (design) + comments (wording) |
| Nonobvious Code | Meaning or behavior isn't clear on a quick read | complexity |

### 3.1 Shallow Module

The cost of a unit is its interface; the benefit is what it does. A shallow unit costs about as much to learn as it saves.

**In a diff:** a helper whose name and signature already say everything the body does; a type with many tiny methods, each one trivial step; many small types split to meet a size target, so the system has more interfaces than it needs.

```rust
// Worse: the interface is the implementation
fn add_null_value_for(attributes: &mut HashMap<String, Option<String>>, name: String) {
    attributes.insert(name, None);
}

// Better: inline it, or give the type a method that hides real work
impl Attributes {
    /// Records `name` as present with no value; a later `set` replaces it.
    pub fn declare(&mut self, name: AttributeName) { … }
}
```

```rust
// Worse: callers assemble three pieces for the common case
let file = File::open(path)?;
let buffered = BufReader::new(file);
let mut records = RecordReader::new(buffered);

// Better: the common case is one call; buffering is the default
let mut records = RecordReader::open(path)?;
```

**Limits:** some small pieces are unavoidable. Small private functions are fine when each one reads on its own (see 3.8).

### 3.2 Information Leakage

A design decision (a format, a rule, an ordering, a storage shape) is known in more than one module, so changing it means changing all of them. The worst kind is hidden: two modules agree on something no interface shows.

**In a diff:** encoder and decoder in different files that both know a byte layout; the same constant or rule copied into two modules; a caller that must know how a callee stores data.

```rust
// Worse: writer.rs and reader.rs both know the record layout
// writer.rs
buf.put_u32(key.len() as u32); buf.put_slice(key); buf.put_u64(timestamp_ms);
// reader.rs
let key_len = buf.get_u32() as usize; let key = buf.copy_to_bytes(key_len); let timestamp_ms = buf.get_u64();

// Better: one module owns the layout
mod record_codec {
    pub fn encode(record: &Record, buf: &mut BytesMut) { … }
    pub fn decode(buf: &mut Bytes) -> Result<Record, DecodeError> { … }
}
```

**Fix options:** merge the pieces that share the knowledge, or move the knowledge into one new module — only if it gets a simple interface; otherwise the hidden leak just becomes a visible one.

### 3.3 Temporal Decomposition

Code split by the order things happen (read, then process, then write) instead of by what each piece must know. When two phases need the same knowledge, it ends up in both.

**In a diff:** modules or functions named for phases (`fetch` / `parse` / `store`) that share knowledge; callers that must call several pieces in a fixed order to finish one operation.

```rust
// Worse: phases as modules; both understand the framing
let raw = fetch::read_frame_bytes(&mut socket)?; // parses the length prefix to know where to stop
let frame = parse::frame(&raw)?;                 // parses the same prefix again

// Better: split by knowledge
let frame = frame_reader.next_frame()?;          // owns the framing end to end
```

**Limits:** splitting by phase is fine when the phases use entirely different information.

### 3.4 Overexposure

To use the common feature, callers must learn or set rarely used ones. The best features are the ones callers get without knowing they exist.

**In a diff:** a constructor where most arguments are the same at every call site; a required argument the module could derive from data it already has; an option nearly everyone wants that must be asked for.

```rust
// Worse: every caller chooses things almost nobody needs to choose
let writer = SegmentWriter::new(path, 64 * 1024, FsyncPolicy::OnClose, Compression::None)?;

// Better: common case simple; rare settings behind a separate path
let writer = SegmentWriter::create(path)?;
let writer = SegmentWriter::builder(path).fsync(FsyncPolicy::EveryWrite).build()?;
```

**Limits:** never hide what callers need for correctness — durability, ordering, failure, threading. Hiding it gives a false abstraction: simple-looking, but callers get it wrong.

### 3.5 Pass-Through Method (and other same-abstraction layers)

A method that only forwards to another with the same or nearly the same signature adds an interface and no functionality. Usually the responsibility between the two types is unclear.

**In a diff:** public methods that forward to a field; a wrapper type that re-exports most of an inner type; two adjacent layers that offer the same abstraction.

```rust
// Worse: Orders adds an interface and no functionality
impl Orders {
    pub fn insert(&self, order: Order) -> Result<(), StoreError> { self.store.insert(order) }
    pub fn get(&self, id: OrderId) -> Result<Order, StoreError> { self.store.get(id) }
}
```

**Fix options:** let callers use the lower type and drop the wrapper; move work so the two types stop calling each other; merge them.

**Not this flag:** a dispatcher that chooses which handler runs (a router); several implementations of one trait. Both add real functionality.

**Wrappers that add one feature:** usually shallow. First consider: add the feature to the underlying type (if most callers want it); put it in the one place that needs it; merge it into an existing wrapper; build it standalone without wrapping.

**Interface should differ from storage:** if the public API mirrors internal storage, callers redo the higher-level work themselves.

```rust
// Worse: text stored as lines, and exposed as lines; every caller splits and joins
fn line(&self, index: usize) -> &str;
fn replace_line(&mut self, index: usize, text: String);

// Better: callers think in ranges; splitting lines is the module's job
fn insert(&mut self, at: Position, text: &str);
fn delete(&mut self, range: Range<Position>);
```

**Pass-through variables:** a value threaded through a chain of functions that only hand it on.

```rust
// Worse: only fetch() uses deadline
fn handle(request: Request, deadline: Instant) -> Response { plan(request.query, deadline) }
fn plan(query: Query, deadline: Instant) -> Plan { fetch(query.key, deadline) }
```

Options: put it in an object both ends already share; store a context object in each major component at construction, so it appears only in constructors. Avoid globals (two instances can't coexist, which breaks tests). A context has costs too: it can become a grab-bag with hidden dependencies; keep its fields few, explained, and immutable.

### 3.6 Repetition

The same, or nearly the same, nontrivial code appears in several places: the right abstraction is missing.

**Fix options:** extract a function when the snippet is long and the new signature stays simple; otherwise restructure so the code runs in one place.

```rust
// Worse: the same release-and-log before every early return
if !is_valid { lease.release(); warn!(%id, "rejected"); return Err(Rejected::Invalid); }
…
if is_expired { lease.release(); warn!(%id, "rejected"); return Err(Rejected::Expired); }

// Better: the cleanup runs in one place
let outcome = check(&request);
lease.release();
if let Err(reason) = &outcome { warn!(%id, ?reason, "rejected"); }
outcome
```

**Limits:** a one- or two-line snippet may not be worth a function; a snippet that touches many locals needs a complex signature, which removes most of the benefit.

### 3.7 Special-General Mixture

A general mechanism contains code for one particular use. The mechanism gets harder, and every change to that use now touches the mechanism. Over-specialization is one of the largest sources of complexity.

**Somewhat general-purpose:** what the module does reflects today's needs; its interface should not be tied to them. Ask:

- What is the simplest interface that covers all current needs? Fewer methods with the same power usually means more general — unless each method grows many arguments.
- In how many situations will this method be used? Built for one use → probably too special.
- Is it easy to use for current needs? If callers need lots of extra code, it went too far.

**Where special code goes:** up into the layer that owns the feature (the UI owns "backspace"; the text module offers "delete range"), or down into implementations of one general interface (drivers behind "read block" / "write block").

```rust
// Worse: a general retry loop knows one caller's error
pub fn retry<T>(policy: &RetryPolicy, mut op: impl FnMut() -> Result<T, Error>) -> Result<T, Error> {
    // … if error.is_rebalance_in_progress() { continue } …
}

// Better: the loop stays general; the caller supplies its rule
pub fn retry<T, E>(
    policy: &RetryPolicy,
    mut op: impl FnMut() -> Result<T, E>,
    is_retryable: impl Fn(&E) -> bool,
) -> Result<T, E> { … }
```

```rust
// Worse: undo logic for every kind of change lives in the text module
impl Text { fn undo(&mut self, ui: &mut Ui) { /* text entries here, selection/cursor entries call back into ui */ } }

// Better: a general history; each kind of change knows how to undo itself; the UI decides grouping
trait Action { fn undo(&mut self); fn redo(&mut self); }
struct History { actions: Vec<Box<dyn Action>>, … }
impl History { fn push(&mut self, action: Box<dyn Action>); fn end_group(&mut self); fn undo(&mut self); }
```

**Special cases in code:** design the normal case so it handles the edge case with no extra code.

```rust
// Worse: every user checks for "no selection"
struct Editor { selection: Option<Range<Position>> }

// Better: an empty range is the normal case
struct Editor { selection: Range<Position> } // start == end means nothing selected
```

**Limits:** separate general from special code within one mechanism. Special code for mechanism A may live with general code for mechanism B when it belongs with B (undo actions for text edits live in the text module, not in `History`).

### 3.8 Conjoined Methods

Two pieces of code that can each be understood only by reading the other. Applies to functions, and to any two separated places that must be read together.

**Depth before length:** once a function is a few dozen lines, making it shorter rarely helps readability. Make functions deep first, then short enough to read easily. Each function should do one thing and do it completely. Length alone is not a reason to split.

| Split | When it helps |
|-------|---------------|
| Extract a subtask | The child reads without the parent, and the parent reads without the child's body. Often the child is general-purpose. The best kind of split. |
| Two functions visible to callers | The original did several unrelated things through a complex interface, and most callers need only one of the new functions. Rare. |
| Shallow pieces | Callers must call each piece in order and pass state between them. Don't. |

```rust
// Worse: split for length; parse_body silently relies on parse_header leaving self.pos at the body
fn parse(&mut self) -> Result<Message, ParseError> {
    self.parse_header()?;
    self.parse_body()
}

// Better: the dependency is in the signature (or the two stay one function)
fn parse(bytes: &[u8]) -> Result<Message, ParseError> {
    let header = parse_header(bytes)?;
    parse_body(&header, &bytes[header.len..])
}
```

```rust
// Worse: one-line helpers, each used once; readers jump between call site and helper to check what is logged
fn log_open_error(request: &Request, peer: SocketAddr, error: &io::Error) { warn!(…) }

// Better: log where the error is detected
Err(error) => { warn!(request_id = %request.id, %peer, %error, "cannot open connection"); return None; }
```

**Not a finding:** a long function with a simple signature that reads top to bottom.

**When joining helps:** it replaces two shallow functions with one deep one, removes duplication, removes an intermediate value passed between them, or puts knowledge in one place.

### 3.9 Hard to Pick Name — design signal

If no short, precise name fits, the thing probably has no single clear purpose.

**In a diff:** names joined with `and` / `or` / `maybe` / `unless`; one variable that holds different kinds of value at different times (logical vs physical block numbers under one name is a classic source of data-corrupting bugs).

```rust
// Signal: three jobs in one function
fn flush_and_maybe_commit_unless_paused(&mut self) { … }
// Fix the design (split the jobs, or make "paused" a state where flush can't be called), not the name
```

This stage reports the design problem; the naming stage owns the wording.

### 3.10 Hard to Describe — design signal

A doc comment should be short and complete. If a complete one must be long, or must describe how the implementation works, the interface is too complex or the unit is shallow. A hard-to-write comment is an early warning.

**In a diff:** a public doc that needs paragraphs or a list of special cases; a field comment that explains several meanings.

**Limits:** a short comment only signals a simple interface if it is also complete.

### 3.11 Nonobvious Code

Obvious code is code where a reader's first quick guess about its meaning is right.

| Makes code less obvious | Prefer |
|-------------------------|--------|
| Callbacks / event handlers: you can't tell which one runs or when | Each handler's doc says when and on which thread it runs |
| Tuples or generic containers for values with meaning | A named struct |
| A declared type that hides an actual type whose behavior matters | Name the real type, or say why it is hidden |
| Code that breaks expectations (a constructor that starts threads; work that continues after `main` returns) | Document it at the surprise point |

```rust
// Worse: the caller sees .0 and .1
fn vote_status(&self) -> (u64, bool)

// Better
struct VoteStatus { term: u64, voted_this_term: bool }
fn vote_status(&self) -> VoteStatus
```

Order of fixes: first reduce what the reader must know (better abstraction, fewer special cases); then follow conventions the reader already knows; then add names and comments.

### 3.12 Flags owned by other stages

- **Comment Repeats Code:** could someone who has never seen the code write this comment from the code next to it? → comments stage.
- **Implementation Documentation Contaminates Interface:** public docs that explain internal mechanisms instead of what the result means to callers → comments stage.
- **Vague Name:** `count` (of what?), `status` for a bool, `x` / `y` for positions in a file → naming stage.

---

## 4. Errors and special cases

Error handling is one of the largest sources of complexity. Every error a function can return is part of its interface, and handlers are the least tested code. Goal: **reduce the number of places where errors must be handled.** Before adding an error, ask whether the caller can do anything different with it; if every caller would ignore or pass it on, it should not exist.

Prefer in this order:

| Technique | What | Rust shape |
|-----------|------|------------|
| **Define the error out** | Redefine the operation so the "error" input is a normal case: "ensure X is gone" instead of "delete X"; a range read returns the overlap, empty when none | `remove(&mut self, id) -> Option<Session>`; `samples_between(start, end) -> &[Sample]` |
| **Mask it** | Handle it low down so higher levels never see it | Retries and reconnects inside the client, not at every call site |
| **Aggregate it** | One handler for many errors, high up; the message is made where the error is raised | `?` through the request path; one `impl IntoResponse for RequestError` at the edge instead of `map_err` + response building in every handler |
| **Just crash** | For errors that are rare and that nothing can sensibly handle, abort with a clear message | `expect("why this cannot happen")` on a broken invariant, rather than an error variant nobody can act on |

```rust
// Worse: callers must handle "already gone", and most just ignore it
fn delete_temp_dir(&self, path: &Path) -> Result<(), DeleteError> // Err(NotFound) when absent

// Better: the operation ensures the directory is gone; absent is success
fn ensure_temp_dir_removed(&self, path: &Path) -> io::Result<()> // only real I/O failures
```

```rust
// Worse: every handler builds its own error response
let quantity = match params.get_u32("quantity") {
    Ok(q) => q,
    Err(e) => return (StatusCode::BAD_REQUEST, format!("bad quantity: {e}")).into_response(),
};

// Better: the error carries its message; one place turns errors into responses
let quantity = params.get_u32("quantity")?; // RequestError::BadParameter { name, reason }
```

**Limits:**

- Define away or mask an error only if callers don't need the information. A module that swallows every network error makes robust callers impossible — they can't tell lost messages or failed peers. What is not important should be hidden; what is important must be exposed.
- Crashing depends on the product: a replicated store must recover from a disk error, because recovery is what it sells. Don't turn frequent errors into crashes.
- Keep errors that end one request separate from errors that are fatal to the process.

---

## 5. Pull complexity down

A module usually has more callers than authors; it is better for the authors to take the pain. A simple interface matters more than a simple implementation.

**Configuration pushes complexity up.** Each knob makes every caller or operator learn and choose it, and values go stale. Before adding one, ask: can callers really pick a better value than the module can compute or measure? If yes, still give a default so most never set it.

```rust
// Worse: every deployment must pick these
pub struct ClientConfig { pub retry_interval_ms: u64, pub batch_linger_ms: u64, pub max_in_flight: usize }

// Better: derive what the module can measure; keep a knob only where callers know better
pub struct ClientConfig { pub max_in_flight: Option<NonZeroUsize> } // None = sized from observed latency
```

**Limits:** pull down only when the complexity is closely related to the module's job, it simplifies many callers, and it simplifies the module's interface. Moving one caller's special behavior into a general module (a "backspace" method in a text module) is leakage, not depth.

---

## 6. Together or apart?

Splitting has costs: more interfaces, code to coordinate the pieces, related code far apart, duplication.

Bring code together when the pieces:

- share information (both depend on one format or rule);
- are used together in both directions (a cache always uses a hash map, but hash maps are used without caches → keep apart);
- overlap in concept under a simple common name;
- can't be understood one without the other;
- would get a simpler interface combined (no intermediate value handed between them; a default replaces a choice);
- duplicate each other.

Keep general and special code apart. Things that look related are not enough: a cursor and a selection merged into one object with flags made both use and implementation harder than two plain positions.

Choose the structure with the best information hiding, the fewest dependencies, and the deepest interfaces.

---

## 7. Design it twice

For each major decision, sketch two or more really different options — even if one seems obviously right. Sketch only the main methods. Compare on ease of use for callers first, then simplicity, generality, efficiency. The best result may combine options; if none is good, use their weaknesses to find another. For a type, this takes an hour or two.

Example: a text module designed by lines, by characters, and by ranges. Lines make callers split and join; characters make callers loop and are slow. If callers do text work around the module, the module is in the wrong shape → ranges.

**Review use:** for a new public interface or major structure, ask whether a different alternative was considered and why it lost. Suggest one concrete alternative when callers visibly work around the new interface.

---

## 8. Comments as a design tool

- Without comments, an interface can't hide complexity: if callers must read the body to use a function, there is no abstraction. Good comments are part of the design, not a failure of the code.
- Writing the interface comment before the body is the cheapest design check: if it is hard to write simply and completely, the design is not ready.
- An interface comment holds: behavior as callers see it; each argument and the return value precisely, with constraints; side effects; errors; preconditions (keep them few).
- A decision that spans modules is documented once, where people will look (for example, next to the enum where new values are added, listing every place that must change), with pointers elsewhere.

Comment wording belongs to the comments stage. This stage uses comments as evidence: a hard-to-write or implementation-heavy interface comment means a design problem.

---

## 9. Strategic, not tactical

- Tactical work aims only to make the change work; each shortcut adds a little complexity, and they pile up. Strategic work aims for a good design that also works.
- After a change, the code should look as if it had been designed with that change in mind from the start. If a change doesn't make the design a little better, it probably makes it worse.
- Grow software by abstractions: when a feature needs a new concept, design that concept cleanly then.

**Limits:** deadlines and cross-team breakage can justify a quick fix; then pick the cleanest option that fits and plan the cleanup.

**Review use in this pack:** judge the change itself — did it bolt on a special case, flag or dependency where fitting it into the design was cheap? Do not demand cleanup of neighbouring code; this pack keeps edits surgical, so offer adjacent improvements as follow-ups.

---

## 10. Consistency

Similar things are done the same way; different things are done differently. Follow the conventions already in the code. Don't introduce a second way of doing something unless there is new information and the gain is worth migrating every old use. Enforce syntax with tools, not review comments.

**Limits:** forcing different things into one name or pattern is confusing — consistency only helps when "looks like X" really means "is X".

---

## 11. Common practices, judged by complexity

- **Shared-state inheritance** creates dependencies between parent and children; prefer composition. In Rust: trait default methods are fine; a trait whose defaults need many getters into the implementer's state is the same coupling.
- **Getters and setters** are shallow and expose representation; ask what operation the caller needs instead.
- **Design patterns** help when the problem has that shape; over-applying them adds ceremony.

---

## 12. Performance

- Know what is expensive (network round trips, storage I/O, allocation, cache misses) and pick the naturally efficient option when it is just as simple (a hash map over an ordered map unless order is needed).
- Add complexity for speed only with measurements; if a change brings no measurable gain and no simplification, back it out.
- On a known hot path: first look for a fundamental fix (a cache, a better algorithm). Otherwise design around the common case — ideally one check up front that catches every special case, with special cases handled off the hot path.
- Shallow layers cost time as well as clarity: each layer adds calls and repeated checks. Simpler, deeper code is usually faster.

Budgets and hot-path costs of a component API are checked in the architecture stage. This stage flags complexity added for unmeasured speed, and special cases or shallow layers on a known hot path.

---

## 13. Decide what matters

Structure code around what matters; keep what doesn't from affecting the rest.

- Look for leverage: one solution that solves many problems, or an invariant that predicts behavior in many situations.
- Make as little matter as possible: fewer required parameters, defaults for common use, errors handled low down, configuration computed instead of chosen.
- Make what matters visible: in interface docs, names, and the parameters of heavily used functions.
- Two mistakes: treating too many things as important (arguments most callers don't care about — a common cause of shallow modules); missing something important (it gets hidden, or callers keep recreating it).

---

## 14. Review checklist

For each new or changed unit:

1. **Depth:** is the interface much simpler than what it does? Any pass-through method, pass-through variable, or wrapper that only forwards?
2. **Leakage:** is any decision (format, rule, ordering, storage shape) known in two places, including places no interface links?
3. **Structure by knowledge:** are pieces split by when they run while sharing knowledge?
4. **Common case:** can the common use be written without learning rare options? Could the module derive a required value or setting itself?
5. **General vs special:** does a general mechanism contain one caller's special case? Do special-case `if`s appear where an empty or default value would do?
6. **Errors:** for each new error: can the operation be redefined so it disappears, handled lower down, handled with many others in one place, or should it abort? Is anything swallowed that callers need?
7. **Split / join:** can each piece be read alone? Was something split for length only? Is code repeated?
8. **Signals:** is a name hard to pick, or a complete doc long? Treat it as a design problem.
9. **Obvious:** tuples with meaning, hidden side effects, callbacks with no "when called"?
10. **Change shape:** was the change bolted on (special case, flag, extra dependency) where fitting it in was cheap? For a new interface, was a different design considered?

Report each finding with the red flag name (or principle), the symptom it causes (change amplification, cognitive load, unknown unknowns), the evidence at path:line, and the smallest change that removes it.
