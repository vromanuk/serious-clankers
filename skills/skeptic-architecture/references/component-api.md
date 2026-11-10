# Component API (provided face)

**Question:** Is this component’s public face the smallest complete set of assumptions a caller needs to finish the job?

Applies to **any** job component (in-process struct, library crate root). Not HTTP, URLs, or UI.

Construction (passed-in, fakes, hot path, “slots stay inside”) stays in `components.md` § High-level API design. This file is the **provided** face: which methods exist, at which level, and whether a stranger can finish the job from the call site.

**Trigger:** the diff changes provided `pub` — job-struct methods, types callers name, library crate root exports. Private / `pub(crate)` helpers only → `component-api: none`. Binary-private structs still get the walk; principle 9 (don’t change meaning) is **stricter** for library `pub`.

---

## Principles

One idea each. Sources at the end are provenance, not extra checklists.

### 1. The face is an obligation

The API is everything a caller must assume to be correct: signatures, meaning, errors, order, side effects, units, what empty means. Callers cannot (and should not) read the body. If they must, there is no interface.

### 2. Hide decisions that will change; publish meaning that stays

Decompose around **secrets** (storage, provider, retry, byte layout) — not around flowchart steps (`load` → `validate` → `write`). Public methods are the assumptions that remain when the secret changes. Exporting the pipeline publishes the implementation.

### 3. Design from the caller’s job, outside-in

Start from what a stranger is finishing, not from tables or generated clients. Write a short sample call site first. If that code is ugly, the face is wrong.

“Does one thing well” is this: a purpose the **caller** can say in one sentence, and the method actually finishes it. Splitting for change-pressure inside the crate is SOLID, not this walk.

### 4. Smallest complete surface

| Too small / shallow | Too large |
|---------------------|-----------|
| Caller still sequences three `pub` functions for one job | Methods with no use case; knobs everyone must learn |

**Complete:** every real use case is one call (or an obvious pair), including the failure path. **Minimal:** every public name has a use case; when in doubt, leave it out — you can add, you cannot take back.

A helper that *adds* a method so the caller does not build an index is completeness. Extra constructors for a future that does not exist are weight.

Depth is the mechanism: the interface should be much simpler than the work behind it. Pull mess down (retries, empty checks, step order).

Interior composition (one function per task) is **conventions**. This walk only asks whether those leaves stayed **private**. Full letter example: `../../skeptic-conventions/references/write-code.md` § Composition.

```text
# Public job — one complete use case
letter(first, last, signature):
  join(opening(first), body(first, last), closing(signature))

# Private tasks
opening / body / closing
```

| Shape | Verdict |
|-------|---------|
| Only `letter` is public | Complete face; interior composed |
| `opening` / `body` / `closing` also `pub` | Leaked composition; caller becomes the composer |
| Everything inline in `letter` | Monolith — conventions, not this walk |

### 5. One abstraction level on the face

Callers work in job terms (`allocate`, `can_accept_rows`, `issue_invoice`). Execution terms (slots, parts, sensor millilitres, SQL rows) stay inside. If two levels must meet, that is a **private** interior, not a second public method that leaks the lower level.

### 6. Local reasoning at the call site

A stranger writes correct code from names and types, without docs for the happy path **or** the failure path. Hidden assumptions (order, units, “empty means missing”, silent defaults) **are** the API whether you wrote them down or not. Make them small, true, and visible.

Work still running after a method returns is a hidden assumption too: take the task owner as an argument or return the handle (skeptic-async `lifetimes-and-load.md` § 2.7).

### 7. Hard to misuse

Illegal states should be hard to represent. Don’t make the client do work the component can do. Don’t pass the buck (a thin wrapper that still dumps every decision on the caller).

Required collaborators are **also** the API: a constructor that demands `aws_sdk_s3::Client` publishes S3. Pass in a role (`ObjectStorage`), not a vendor. Detail: `components.md` § External dependencies.

### 8. Callers retry, skip, and get it wrong

They will call twice, crash between steps, pass an empty list, cache a value too long. Operations they will repeat must be safe to repeat (naturally, or with an explicit token / “ensure” / draft-then-confirm). Unbounded “give me everything” has a hidden assumption that the world is small. Data that goes stale must say until when.

### 9. You may add meaning; you may not silently change it

For anything others already call: additive change; the old method keeps its meaning (it may become the special case of a more general one). Observable behavior (event order, “empty means not found”) is contract whether documented or not. Stricter for library `pub`; looser for a struct only this binary uses.

### 10. The component is the unit

One job, one namespace, one provided face (prefer one struct + use-case methods), private interior, data owned here, wired at the app edge. A slice with no dedicated face is not a component — every internal type becomes accidental API. Nesting is allowed; nested parts do not become public just because they exist. Layout detail: `components.md`.

### 11. Consider a batch as the base case

A consideration, not a rule (matklad, "Push Ifs Up And Fors Down"; TigerBeetle's API takes transfers in batches). When callers have many items — records, transfers, edits — a face that takes one item at a time pushes the loop, and every per-call cost, onto them. A face that takes a batch lets the component pay fixed costs once, choose the order, and check everything before changing anything. A single item is a batch of one.

A batch face must say what happens when some items fail, because that choice is visible to every caller. Two common answers: **one result per item** (later items still run), or **all or nothing**. Either way, say when the batch is durable relative to its effects.

```rust
pub struct Bank<J: Journal> {
    ledger: Ledger,
    journal: J,
}

impl<J: Journal> Bank<J> {
    /// Applies a batch: one result per transfer, in order, and one journal write for the whole
    /// batch. The batch is durable before any balance changes, so if the write fails the ledger
    /// is untouched and the whole batch can be retried.
    pub fn create_transfers(
        &mut self,
        transfers: &[Transfer],
    ) -> std::io::Result<Vec<Result<(), TransferError>>> {
        let batch = self.ledger.prepare(transfers);
        self.journal
            .append_durably(&select_accepted_transfers(transfers, &batch.results))?;
        Ok(self.ledger.commit(batch))
    }
}
```

`Ledger::prepare` works out every result as if the earlier transfers had been applied, without changing the ledger; `commit` applies them only after the journal write succeeds. The tests that pin the face down:

- a failed transfer does not stop the rest of the batch;
- a later transfer sees the balance left by an earlier one;
- the journal is written once per batch;
- a failed journal write leaves every balance unchanged.

Not every face needs this: a single-item face fits when items arrive one at a time and nothing is shared between them. The batching that happens *inside* an event loop is in `skeptic-async/references/select-and-tasks.md` § 2.7.

---

## Walk

On each changed provided surface, in this order:

1. **Job in one sentence.** Each new or changed `pub` method: what is it for? No answer → it does not belong.
2. **Sample caller** (about 10–20 lines) for the main use case. Still sequencing three or more `pub` functions, or naming slots / rows / sensors → incomplete or wrong level.
3. **Smallest complete.** Each `pub` name ↔ a use case. Common case has no extra knobs. Missing helper and extra knob both fail.
4. **Assumptions visible.** Mutating looks mutating; units in the type or name; empty vs missing vs error vs omitted are different; defaults do not hide mistakes; validate, don’t clamp; errors the caller can act on (unresolvable first); lists bounded; repeatable if they will retry.
5. **If this `pub` already existed:** meaning unchanged, or the old method kept as the special case of a more general one.

Then: `component-api: ok` (job + one-line sample caller) or findings. LETTER options when the fork is “add a use-case method” vs “keep steps public.”

Diff does not change a provided face → `component-api: none — <why>`.

---

## Examples

### Letter → `allocate` (principles 3, 4, 10)

```text
# Bad public — leaked composition / flowchart
allocation.load_batches(sku)
allocation.pick_batch(line, batches)
allocation.save_allocation(line, batch)

# Good public — one complete job
allocation.allocate(order_id, sku, qty) -> BatchRef | OutOfStock
# pick_batch stays private thinking; store stays private
```

Sample caller that must stay short:

```text
let batch = allocation.allocate(order, sku, qty)?;
```

If that sample still has three calls, the face is incomplete or too shallow.

Same finding as “stray public helpers” in `components.md`. Report **once**, here, when it is the job’s provided face.

### Check-engine (principles 4, 10)

Public: `schedule(check)`, `results(query)`.  
Private: queue, database, runner.

Bad: `pub` on `enqueue_raw` / `insert_result_row` so callers “compose” the engine.

### One level — coffee ready, export slots (principle 5)

```text
# Bad: caller compares two levels
recipe.volume vs order.volume_poured   # “is it done?”

# Good: job term
order.status() -> Preparing | Ready | Failed
```

```text
# Bad: execution leaked
window_export.try_start_upload() -> AllUploadersBusy

# Good: job term
window_export.can_accept_rows()
window_export.add_rows(...)
```

`AllUploadersBusy` is also containment in `components.md`. Report **once**, here (wrong level on the provided face).

### Complete vs extra knobs (principle 4)

```text
# Incomplete: caller does the job
recipes = coffee.all_recipes()
machines = coffee.all_machines()
index = build_geo_index(recipes, machines)
hits = index.query(here)

# Complete
offers = coffee.search(here, recipe?)
```

```text
# Not minimal
SegmentWriter::new(path, buf, fsync, compression, retry, checksum, ...)

# Minimal common case; rare options on a second path
SegmentWriter::create(path)
```

Overexposure of knobs on *any* unit is also complexity. On the **job’s public constructor / methods**, report here.

### Explicit and typed (principles 6, 7)

```text
# Implicit
order.canceled = true
stats = orders.get_stats()        # scans forever
duration: 5000                    # of what?
status: true
connect(url: &str)

# Explicit
order.cancel()
orders.aggregated_stats(from, to)
duration_ms: 5000
is_finished: true
connect(url: Url)
widget(Size::Small, Shape::Round)  # not (true, false)
```

`get_*` / HTTP GET that writes → hard-rule `HR-api-get-mutates` (stage 10). Bare public `status: bool`, units, double negation, name↔payload → `HR-api-*` / naming. Here: the **shape** (mutating looks mutating; types carry meaning). If the same line is an `HR-api-*`, note it and leave the id to hard-rules.

```text
# Required side
WindowExport { s3: aws_sdk_s3::Client }     # secret leaked into the face
WindowExport<S: ObjectStorage> { storage: S }  # role; see components.md
```

### Empty, validate, repeat (principles 6, 8)

```text
search(query) -> Ok([])           # nothing matched — not NotFound
cart(id) -> Err(NotFound)         # missing cart is not Ok(empty items)

# omitted field ≠ empty vec (don’t treat recipes: [] as “no filter”)

put(latitude: 100) -> Err(...)    # don’t clamp to 90
missing location -> Err(...)      # don’t default to a silent “somewhere”

create_order(idempotency_key, ...)
# or ensure_order(...) / draft then confirm
```

Silent empty success when empty is a **product error** is also type-driven. Report once: type-driven if the *type* can force it; here if the *operation meaning* is wrong (search vs get-by-id).

### Compatibility (principle 9)

```text
machine.prepare_lungo(volume)                 # keep; meaning unchanged
machine.prepare(recipe, volume, options)      # added general job
# prepare_lungo still means lungo, same defaults
```

---

## Assumptions on the face (principle 6–8, compact)

Not a second walk. Use when step 4 needs a name for the miss.

| Assumption | Prefer | Not |
|------------|--------|-----|
| Side effects | Mutating names look mutating | `get_*` / field-set that cancels |
| Units | In the type or the name | Bare `5000`, money as `f64` |
| Empty vs missing | Typed: `Ok([])` vs `Err(NotFound)` vs omitted | One `Vec` for both; empty filter = no filter |
| Defaults | Missing required input is an error | Silent “somewhere” / clamp |
| Errors | Caller can act; unresolvable first; no two that bounce | Empty `Err`; price-changed then “cannot order anyway” |
| Size | Lists / batches bounded on the face | Unbounded `list_all()` as the only read |
| Repeat | Create/ensure is safe to retry | Bare `create` that doubles on retry |
| Stale data | `valid_until` (or equivalent) on the value | A price with no lifetime |

---

## Walked examples (collisions)

Same five steps on the gallery. Owner = where the finding is filed if several stages could claim it.

| Surface | Walk | File under |
|---------|------|------------|
| `allocation.allocate(...)` only public | Job clear; sample caller one line; complete | `component-api: ok` |
| `load_batches` / `pick_batch` / `save_allocation` public | Sample caller sequences three; leaked letter-leaves | **here**, not complexity shallow-module, not conventions |
| `opening` / `body` private, `letter` public | Interior composition | conventions `ok`; **not** this walk |
| `try_start_upload` → `AllUploadersBusy` | Wrong level (slots) | **here**, not a second containment finding |
| `get_stats()` that writes or scans forever | Mutating/expensive looks like a read | `HR-api-get-mutates` / naming; this walk may note shape once |
| `search` → `Ok([])` vs `cart` → `Err(NotFound)` | Empty vs missing | **here** for operation meaning; type-driven if a non-empty type would force it |
| `prepare` added beside unchanged `prepare_lungo` | Meaning kept | `ok` |
| Private helper rename only | No provided `pub` | `component-api: none` |

## Not a finding here

| Symptom | Owner |
|---------|--------|
| Interior split / monolith *inside* a private method | conventions (`write-code.md`) |
| Conjoined private helpers, pass-throughs not on the job face | complexity |
| CUPID as counterweight to “split / add a trait” | SOLID (`cupid.md`) |
| Name spelling, `is_` prefixes, `HR-api-*` ids | naming / hard-rules |
| Passed-in, fakes, hot path, memory limit | `components.md` construction checks |
| Empty-as-error that a non-empty type would force | type-driven |
| Timeouts, retry *implementation*, authn | reliability / security |
| HTTP paths, status codes, UI widgets | **never this file** |

If layout (`components.md` stray helpers) and this walk would both fire on the same exported steps, **this walk owns the finding**.

---

## Review one-liners

- `component-api: ok` — e.g. “job is allocate stock; sample caller is `allocation.allocate(...)`; `pick_batch` private.”
- `component-api: none` — e.g. “diff only renames a private helper.”
- Finding: “root exports `load_batches` / `pick_batch` / `save_allocation`; sample caller sequences them — fold into `allocate` (letter: `opening` is not public).”
- Finding: “`try_start_upload` → `AllUploadersBusy` — caller manages slots; job term is `can_accept_rows`.”
- Finding: “caller lists all recipes and machines and builds a geo-index — add `search`.”
- Finding: “`prepare_lungo` now takes `options` that change default volume — that’s a meaning change; add `prepare` and leave `prepare_lungo`.”
- Not a finding: “`opening` is a private function” — that *is* composition.
- Not a finding: “no `/v1/` on the method” — not a wire API review.

---

## Sources (provenance)

Parnas (interface = assumptions; hide decisions that change). Bloch (easy to use / hard to misuse; smallest but complete; don’t make the client do the library’s job). Qt / Blanchette (minimal *and* complete). Ousterhout (deep module; pull complexity down). Konstantinov, *The API* (obligation; pyramid; explicit face; empty vs missing; extend by abstracting) — recast for a job struct, not HTTP. CUPID (small surface, outside-in purpose, predictable) — properties; the SOLID stage owns the acronym. Hombergs / Brown (dedicated face vs internal). Henning (caller’s perspective). Meyer (commands vs queries). Hickey (simple = not complected). Rust API Guidelines (`C-CUSTOM-TYPE`, `C-VALIDATE`). HtDP composing functions — letter in `write-code.md`.
