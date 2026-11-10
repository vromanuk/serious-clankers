# Rust craft and control flow

## matklad — push ifs up, fors down

Source: [Push Ifs Up And Fors Down](https://matklad.github.io/2023/11/15/push-ifs-up-and-fors-down.html) (matklad, 2023).

### Push ifs up

**Idea:** Decide early (caller / outer function); leave the tight loop as straight-line work on already-valid data.

```rust
// Prefer: caller resolves Option / Result / mode, then calls a simple body
fn handle(req: Request) {
    let user = match req.user {
        Some(u) => u,
        None => return reject(),
    };
    process(user); // no nested ifs inside process for "missing user"
}

// Avoid: deep nesting inside the hot path for control decisions
```

With all the branching in one function, dead or repeated conditions become visible. One form is the "dissolving enum": a function decides, returns an enum that records the decision, and another function matches on it — the same condition checked twice, once as data. When nothing else needs the enum, decide where the work happens:

```rust
// Worse: `use_cache` is decided in `pick_source`, stored as an enum, and decided again in `read`
enum Source { Cache(CacheKey), Database(Query) }
fn pick_source(request: &Request) -> Source {
    if request.use_cache { Source::Cache(request.cache_key()) } else { Source::Database(request.query()) }
}
fn read(source: Source) -> Rows {
    match source { Source::Cache(key) => read_cache(key), Source::Database(query) => run_query(query) }
}

// Better: one decision, next to the work
fn read(request: &Request) -> Rows {
    if request.use_cache { read_cache(request.cache_key()) } else { run_query(request.query()) }
}
```

Keep the enum when the decision crosses a real boundary — returned from pure code to a shell, sent in a message, or stored.

When reviewing: flag functions that mix policy branching with dense loops when the branch could be lifted.

### Push fors down: the batch is the base case

Hot paths handle many things; the volume is what makes them hot. Make the operation on a batch the basic one, and treat a single item as a batch of one:

```rust
// Worse: the caller owns the loop. Each call pays the per-call cost (a journal write and sync),
// and the first failure stops the batch, so the caller must rebuild per-item results itself.
for transfer in &transfers {
    bank.create_transfer(transfer)?;
}

// Better: one call for the batch, one result per transfer, one journal write
let results = bank.create_transfers(&transfers)?;
```

A batch lets the callee pay fixed costs once (a write, a sync, a lock, a header), choose the processing order, and check the whole batch before changing anything. The two rules compose: decide once, outside the loop, then run straight loops.

```rust
// Worse: the format is checked again for every row, and a per-batch step (the CSV header)
// has nowhere to go
for row in rows {
    match format { Format::Csv => write_csv_row(row), Format::JsonLines => write_json_row(row) }
}

// Better: branch once; each loop is straight, and the CSV header is written once
pub fn render(rows: &[Row], format: Format) -> String {
    match format {
        Format::Csv => render_csv(rows),
        Format::JsonLines => render_json_lines(rows),
    }
}

fn render_csv(rows: &[Row]) -> String {
    let mut out = String::from("account,balance_cents\n"); // the header is written once per batch
    for row in rows {
        out.push_str(&format!("{},{}\n", row.account.0, row.balance_cents));
    }
    out
}
```

These are considerations, not rules: a batch API is worth it when per-call costs are real or the caller would otherwise loop. The component-level version — publishing the batch method on a component's face — is in `skeptic-architecture/references/component-api.md` § 11.

## Assertions: catch bugs where they happen

An **operating error** is expected: a missing file, a timeout, bad input from a user or another service. It is returned (`Result`) and handled. A **bug** is the program contradicting itself: an invariant it relies on is false. No caller can act on it, so the program stops (`assert!`, `expect`, `unreachable!`) close to the cause, instead of carrying corrupt state forward. An assertion turns a silent correctness bug into a visible crash, and gives fuzzing and property tests something to find. The error-design side is in `skeptic-complexity/references/complexity.md` § 4.

Where to assert:

- **Preconditions and invariants the code relies on**, checked where they are established — a constructor that guarantees "the sum of all balances fits in `u64`" lets every later `checked_add` treat overflow as a bug.
- **The same property from two sides.** Check data you produce before writing it, and check it again when reading it back. The two checks differ: your own encoder producing bad bytes is a bug (assert); bytes read from disk or the network can be corrupt, so the reader returns an error.
- **What must not happen, as well as what must.** `if index < count { … } else { … }` handles both sides; a lone `if` leaves the other case unspoken.
- **Relations between constants, at compile time**, so changing one cannot silently break another.

```rust
/// Most transfers in one batch; a full batch must fit in one message.
pub const TRANSFERS_PER_BATCH_MAX: usize = 8_000;

// Checked when the crate compiles: changing one constant cannot silently break the others.
const _: () = assert!(TRANSFERS_PER_BATCH_MAX * TRANSFER_SIZE_BYTES <= MESSAGE_BODY_SIZE_MAX_BYTES);

pub fn encode_batch(transfers: &[Transfer]) -> Vec<u8> {
    // The caller is ours and must split larger batches: an oversized batch here is a bug.
    assert!(transfers.len() <= TRANSFERS_PER_BATCH_MAX, "batch of {} transfers", transfers.len());
    // ...
}

/// Bytes read back from disk can be corrupt or truncated: that is an operating error, so it
/// is returned, not asserted.
pub fn decode_batch(bytes: &[u8]) -> Result<Vec<Transfer>, DecodeError> {
    if !bytes.len().is_multiple_of(TRANSFER_SIZE_BYTES) {
        return Err(DecodeError::PartialTransfer { size_bytes: bytes.len() });
    }
    // ...
}

/// Encodes a batch for the journal and checks the property a reader will rely on.
pub fn encode_for_journal(transfers: &[Transfer]) -> Vec<u8> {
    let bytes = encode_batch(transfers);
    // Our own encoder produced these bytes, so a round trip that differs is a bug. The reader
    // checks the same property again with `decode_batch`, where a failure is corruption.
    debug_assert_eq!(decode_batch(&bytes).as_deref(), Ok(transfers));
    bytes
}
```

How to write them:

- **One condition per assertion:** `assert!(a); assert!(b);` says which one failed; `assert!(a && b)` does not.
- **An implication as an `if`:** `if order.is_filled() { assert_eq!(order.remaining_shares, 0); }`.
- **A message with the values:** `assert!(n <= MAX, "batch of {n} transfers")` saves a debugging session.
- **`assert!` vs `debug_assert!`:** `assert!` stays in release builds — use it when the check is cheap next to the work or guards data that would otherwise be corrupted. `debug_assert!` disappears in release — use it for expensive checks (a full round trip, an O(n) sum) on hot paths; tests and debug builds still run it.
- **Test the assertion** with `#[should_panic(expected = "…")]` when it guards a contract callers could break.

**Flag:** an invariant the code depends on with no assertion and no type that enforces it; a compound assertion that hides which condition failed. A `Result` variant for a state only a bug can produce, and a panic on input from outside the process, are error design: `skeptic-complexity/references/complexity.md` § 4.

## Simple control flow

**Split compound conditions.** A condition that joins several booleans makes the reader work out which combinations reach each branch, and often lumps different failures into one answer. Ask one question per branch, and name every case:

```rust
// Worse: three reasons collapse into one rejection, and the reader must work out
// which combinations reach the else
if market == MarketState::Open
    && order.shares > 0
    && (order.kind == OrderKind::Market || order.limit_price_cents.is_some())
{
    Ok(())
} else {
    Err(Rejection::Invalid)
}

// Better: one question per branch; every combination of kind and price is named
pub fn admit(order: &OrderRequest, market: MarketState) -> Result<(), Rejection> {
    if market == MarketState::Closed {
        return Err(Rejection::MarketClosed);
    }
    if order.shares == 0 {
        return Err(Rejection::EmptyOrder);
    }
    match (order.kind, order.limit_price_cents) {
        (OrderKind::Limit, Some(_)) | (OrderKind::Market, None) => Ok(()),
        (OrderKind::Limit, None) => Err(Rejection::LimitWithoutPrice),
        (OrderKind::Market, Some(_)) => Err(Rejection::MarketWithPrice),
    }
}
```

The worse version also accepted a market order that carries a limit price; naming every pair surfaced that case.

**State conditions positively.** `if index < count` reads the way the invariant is written and matches the loop condition; `if !(index >= count)` and `if index >= count { … }` with no `else` make the reader negate in their head.

**The parent keeps the state; helpers compute.** When one method updates several fields, let it do every assignment and call helpers that only compute values. A reader then sees in one place what changes, and the helpers are pure functions to test:

```rust
// Worse: each helper reaches into `self`; the order of the calls matters
// (`update_cash` reads the fee `charge_fee` just added), and nothing here shows it
pub fn apply_fill(&mut self, fill: Fill) {
    self.charge_fee(fill);
    self.update_cash(fill);
    self.update_shares(fill);
}

// Better: every field this method changes is visible here; the helpers only compute
pub fn apply_fill(&mut self, fill: Fill) {
    self.cash_cents += compute_cash_change_cents(fill);
    self.shares += compute_share_change(fill);
    self.fees_paid_cents += compute_fee_cents(fill);
}
```

## Pass settings explicitly instead of relying on defaults

Write out every library setting that decides correctness or failure behavior, even when it equals today's default. Defaults change between versions and differ between clients of the same system, and an implicit default is invisible in review.

```rust
/// HTTP client for the price service, with every setting that affects failure behavior written
/// out. reqwest has no overall request timeout unless one is set, so the default would let a
/// stuck call wait forever.
pub fn build_price_client() -> reqwest::Result<reqwest::Client> {
    reqwest::Client::builder()
        .connect_timeout(Duration::from_secs(2))
        .timeout(Duration::from_secs(5))
        // Below the price service's 60 s keep-alive, so the pool drops a connection before the
        // server closes it and a request never lands on a dead socket.
        .pool_idle_timeout(Duration::from_secs(30))
        .build()
}
```

The same applies to configuration a library reads from strings. Kafka consumer defaults differ by client: `isolation.level` defaults to `read_committed` in librdkafka (and the Rust `rdkafka` crate built on it) but to `read_uncommitted` in the Java client, so the same service reads aborted transactional messages or not depending on its language. Set what the code relies on:

```text
enable.auto.commit=false        # default true: offsets would be committed before processing finishes
auto.offset.reset=earliest      # default latest: a new consumer group would skip the existing backlog
isolation.level=read_committed  # default differs between clients (see above)
```

**Flag:** a client or consumer built with defaults for timeouts, commit behavior, offset reset, isolation, retries, or delivery guarantees that the code depends on. Not a finding: cosmetic settings (thread names, log formats).

## One copy of each fact

**Do not store what you can derive.** A second copy of a fact must be updated everywhere the first one is, and the one place that forgets makes them disagree:

```rust
// Worse: `push` must remember `free_count += 1`, and `pop` must remember to subtract
struct Slots { free_slots: Vec<usize>, free_count: usize }

// Better: the count is derived from the list, so the two cannot disagree
impl Slots {
    pub fn count_free_slots(&self) -> usize {
        self.free_slots.len()
    }
}
```

**Check a value where you use it.** A condition checked early and acted on later can be stale by then — other code ran in between. Combine the check and the use:

```rust
// Worse: `has_free_slot` was true twenty lines ago; `place_pending_orders` may have taken the slot
let has_free_slot = !self.free_slots.is_empty();
self.place_pending_orders();
if has_free_slot {
    let index = self.free_slots.pop().unwrap(); // can panic
    // ...
}

// Better: the check and the use are the same operation
pub fn take(&mut self) -> Result<usize, BookFull> {
    let Some(index) = self.free_slots.pop() else {
        return Err(BookFull);
    };
    Ok(index)
}
```

Declare a variable where it is first needed and keep its scope small, so there are fewer names in play to confuse.

**Flag:** a stored field that duplicates a value derivable from other fields (a count next to its collection, a total next to its parts) without a measured reason; a check whose result is used after code that can change what was checked.

## Explicit over clever

Prefer **explicit** over clever; small composable pieces; errors you can see; avoid magic globals.

When reviewing:

- Hidden global config that changes behavior without parameters  
- “Clever” macros that obscure the decision  
- Prefer a boring `match` on a decision enum over side effects in a helper named `maybe_do_stuff`

## Density vs cleverness

Dense **local** code that states the domain is fine.  
Dense **abstraction** for one call site is not (small-surface rule).

## Formatter / linter

`rustfmt` + project clippy are the style enforcers. This file is for **shape of control flow and APIs**, not brace placement.

## Sources

- https://matklad.github.io/2023/11/15/push-ifs-up-and-fors-down.html — Push Ifs Up And Fors Down (matklad)
- https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md — TigerStyle (TigerBeetle): assertions, compound conditions, explicit options, variable scope
- https://matklad.github.io/2025/12/29/second-error-model-convergence.html — The Second Great Error Model Convergence (matklad): a separate path for bugs
- https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md — librdkafka configuration (defaults)
- https://kafka.apache.org/documentation/#consumerconfigs — Kafka Java consumer configuration (defaults)
