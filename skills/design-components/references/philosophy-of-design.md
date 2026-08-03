# Deep design (Ousterhout, applied here)

Actionable rules from John Ousterhout, [*A Philosophy of Software Design*](https://web.stanford.edu/~ouster/cgi-bin/book.php) (2nd ed. themes included), mapped onto **this pack’s** component model.

**Not** a book summary. **Not** a second architecture. Use with:

- `guide.md` — component struct + pure-core/shell  
- `skeptic-architecture/references/components.md` — job layout  
- `skeptic-conventions/references/write-code.md` — one function per task (**interior**)  
- `skeptic-comments` — comments as design (adjacent ideas)

---

## The problem: complexity

**Complexity** = anything in the structure that makes the system hard to **understand or change**.

It shows up as:

| Symptom | Meaning |
|---------|---------|
| **Change amplification** | One idea forces edits in many places |
| **Cognitive load** | Too much to hold in mind for a safe change |
| **Obscurity** | Important facts are hidden or surprising |

**Causes:** unmanaged **dependencies** between pieces, and **obscurity** (missing or misleading information).

**Two tactics:**

1. **Eliminate** complexity when you can (simpler shape, fewer special cases).  
2. **Encapsulate** the rest behind a **simple interface** (deep module).

Our public **job struct** is the usual place we encapsulate a job’s complexity.

---

## Seven rules (apply these)

### 1. Prefer deep modules

A **module** here = component, public struct, or important type — anything with an interface outsiders use.

| | **Deep** | **Shallow** |
|--|----------|-------------|
| Interface | Small / simple relative to power | Large or fussy for little benefit |
| Body | Does real work; hides steps | Thin pass-through or re-exports steps |
| Caller cost | Learn a few methods | Learn many helpers and their order |

**In this pack:** one public job struct + few use-case methods is the deep face. Exporting `load_*` / `validate_*` / `write_*` is shallow.

```text
# Shallow — interface ≈ implementation, complexity on the caller
allocation::load_batches(...)
allocation::pick_batch(...)
allocation::save_allocation(...)

# Deep — interface much smaller than the work inside
allocation.allocate(order_id, sku, qty) -> Result<BatchRef, …>
```

### 2. Pull complexity downward

When someone must deal with a messy detail, prefer that **the module** deals with it — not every caller.

| Push up (usually worse) | Pull down (usually better) |
|-------------------------|----------------------------|
| Caller sets 6 knobs every time | Sensible defaults; override only when needed |
| Caller sequences load → check → save | Method does the sequence |
| Caller maps every low-level error | Method returns a few job-level errors |

**Not** “make the implementation pretty at the cost of a hard API.” Prefer a **simple call site** even if the body works harder.

### 3. Hide design decisions (information hiding)

Each module should own a few **decisions** that can change without rewriting callers: storage shape, sort order for stock, how retries work, file format.

**Information leakage:** the same decision is known in two places (two modules know the batch table layout; two places know “allocate earliest ETA first”). Then change amplifies.

| Leak | Hide |
|------|------|
| Callers build SQL for this job’s tables | Only the component’s store path knows tables |
| Callers know step order of a use case | Only the job method knows the order |
| Format string / wire field names spread | One parse/format site behind the face |

Hiding means **callers need not know**, not “never write a private helper.”

### 4. Avoid temporal decomposition of the *public* surface

Splitting **only** by time order of the pipeline (`load_step`, `validate_step`, `save_step` as separate **public** modules) spreads one design across many faces and leaks order.

Prefer modules by **what they own** (job, rules, store), then private steps **inside**.

```text
# Bad public split (temporal)
pub mod load;
pub mod validate;
pub mod save;
// caller must call in the right order

# Good
pub struct Checkout { … }
impl Checkout {
    pub fn place_order(&mut self, …) { /* load, validate, save privately */ }
}
```

Interior may still use ordered private functions. That is composition, not a public timeline API.

### 5. Different layer of abstraction → different interface (no pass-throughs)

A method that only renames one underlying call and adds no value is a **pass-through**: interface cost without hiding power.

| Smell | Fix |
|-------|-----|
| `fn save(&self, x) { self.store.save(x) }` as the only public API | Callers use store only if it *is* the product, or fold save into a real use case |
| Wrapper that re-exports every method of an inner type | Either expose the inner type or a **smaller** intentional face |
| “Facade” that is a 1:1 map of SQL methods | Design use cases, not table verbs |

**Exception:** thin adapters at the **app edge** (HTTP → job method) are fine — their job is wire format, not domain depth.

### 6. General-purpose faces beat special-case knobs (when they fit)

A deeper interface often solves **more than one** call site without growing a forest of flags.

| Special-case pile | More general face |
|-------------------|-------------------|
| `allocate_with_prefer_warehouse_a`, `allocate_ignoring_eta`, … | `allocate(line, Policy { … })` or a small policy type with defaults |
| New method per report format | `export(report_id, Format)` |

Do **not** invent generality you do not need (still “good enough”). When a second special case appears, **consider** generalizing the face instead of cloning methods.

### 7. Define errors out of existence when cheap; otherwise make them obvious

Special cases and error paths are a major source of complexity.

| Prefer when possible | When you must have an error |
|----------------------|-----------------------------|
| API that cannot represent the bad state (types, non-empty, enums) | Few clear error variants on the job method |
| Defaults that avoid “forgot to set X” | Document failure in the public method docs |
| Idempotent “already done” as success when product allows | Do not leak 15 SQL error codes to HTTP |

```rust
// Worse: caller must remember a second check
fn items(&self) -> Vec<Item>; // empty might mean "not found" OR "empty cart"

// Better: type forces the distinction
fn cart(&self, id: CartId) -> Result<Cart, NotFound>;
// Cart::items is never "missing cart"
```

Aligns with this pack’s **type-driven** boundaries (`type-driven.md`).

---

## Balance: deep modules vs “one function per task”

| Rule | Scope |
|------|--------|
| **One function per task** | **Inside** the component — private helpers, pure rules, readable steps |
| **Deep module** | **Public face** — few methods; do not publish every task |

Ousterhout pushes back on “always tiny public methods.” We agree for the **boundary**: small private functions are good; a **shallow public surface** of tiny steps is not.

Clean Code–style “extract until every method is five lines” applied to **exports** often produces shallow modules. Extract **privately**; keep the **struct face** deep.

---

## Strategic vs tactical (short)

| Tactical | Strategic |
|----------|-----------|
| “Make this ticket work with minimal thought to structure” | “Leave the module deeper than you found it when cheap” |
| Adds flags and special cases at the call site | Pulls special cases into the module or types them away |
| Works for a spike | Default for code that will be lived in |

When fixing a bug under time pressure, still avoid **exporting** a new step-helper that leaks the workflow. Prefer one more private branch or one clearer error on the existing method.

---

## Comments as part of the interface (link only)

Ousterhout treats **interface documentation** as part of the module (what callers must know). This pack: **skeptic-comments** — public docs = how to use **and why this shape**; body = non-obvious why. Module template: `comments.md` (role → intuition → why → rules → entrypoints).

When designing a job struct:

- Document **methods** for contracts, failure, and non-obvious ordering **if any remains**.  
- Prefer **removing** ordering from the interface (rule 2–4) over documenting a fragile sequence.  
- Do not use comments to paper over a shallow API — **deepen** the API first.  
- Do not leave “do X, not Y” on the module without the **benefit** or rejected alternative.

---

## Bad / good gallery

### Public surface

```rust
// Bad — shallow: caller is the orchestrator
pub fn load_cart(db: &Db, id: Id) -> Cart { … }
pub fn apply_discount(cart: &mut Cart, code: &str) -> Result<(), …> { … }
pub fn charge(db: &Db, cart: &Cart) -> Result<(), …> { … }

// Good — deep: one handle, methods own the story
pub struct Checkout { db: Db, payments: Payments }
impl Checkout {
    pub fn place_order(&mut self, id: Id, code: Option<&str>) -> Result<OrderId, CheckoutError> {
        let mut cart = self.db.load_cart(id)?;
        if let Some(c) = code {
            apply_discount(&mut cart, c)?; // private pure or private fn
        }
        self.payments.charge(&cart)?;
        self.db.save_order(&cart)
    }
}
```

### Pull complexity down

```rust
// Bad — every caller configures the same pain
store.read_with_options(id, true, false, Some(40), Retry::new(3))?;

// Good — job method (or store helper used only inside the component)
self.store.read_order(id)?; // retries/timeouts inside
```

### Information leak

```rust
// Bad — HTTP handler knows table semantics
sqlx::query("SELECT * FROM batches WHERE sku = $1 AND qty > 0")…

// Good — only Allocation/store knows that
self.store.list_allocatable(sku)?
```

### Pass-through

```rust
// Bad — no depth
impl Allocation {
    pub fn list(&self) -> Vec<Batch> { self.store.list() }
}

// Good — use case or don't wrap
// Callers that need raw list might be wrong; or list is a real product query with rules:
pub fn list_open_batches(&self) -> Vec<BatchSummary> { /* filter, map, authorize */ }
```

---

## Design / review checklist (Ousterhout lens)

1. **Deep enough?** Is the public interface much simpler than the work inside?  
2. **Complexity direction?** Did we pull mess **down** into the module, or push knobs **up** to callers?  
3. **One owner per decision?** Storage, policy, step order — not copied across modules.  
4. **Temporal public API?** Are we exporting pipeline stages instead of a use case?  
5. **Pass-throughs?** Any public method that only renames an inner call?  
6. **Special-case pile?** Third similar flag/method → consider a deeper general face.  
7. **Errors?** Can types remove a class of mistakes? If not, are job-level errors few and clear?  
8. **Interior vs face?** Private helpers small and many is fine; public methods few and deep.

---

## Related pack scars

| Scar | Ousterhout angle |
|------|------------------|
| Stray public helpers | Shallow module; complexity pushed up |
| Handler owns the workflow | App edge doing the module’s job |
| One function per task **exported** | Mis-applied composition; face became temporal API |
| Business rules next to sockets only | No deep pure interior; hard to test and change |

---

## Source note

Ideas above are restated for agent use from Ousterhout’s book (deep modules, information hiding/leakage, temporal decomposition, pull complexity down, pass-through methods, general-purpose interfaces, error complexity, strategic programming, interface comments). Prefer the book for full argument; prefer **this file** for pack decisions.
