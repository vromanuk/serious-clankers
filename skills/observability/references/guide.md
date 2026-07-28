# Observability (Rust-first)

Rules for **writing and reviewing** production telemetry: logs, spans (traces), metrics, and panic reporting.

**Sources:** [Zero to Production in Rust](https://www.zero2prod.com/) — telemetry / “Are we observable yet?” (Luca Palmieri); your Rust Telemetry flashcards (`tracing`, metrics, panics).  
Rust-first tools: `tracing`, `#[instrument]`, OpenTelemetry exporters, `metrics` crate. Ideas apply in any stack (structured logs, spans, metrics).

---

## What observability means

**Observability** means you can ask new questions about a running system **without** having built a special dashboard for that question ahead of time.

You still choose carefully what to emit — full internal state is usually too expensive. Logs/spans/metrics are a **lossy** picture; make each signal earn its cost.

---

## Why we emit signals

- See from the **outside** what is going on **inside** (especially when behavior is wrong).  
- Prefer signals that help **diagnose failures** and **slow paths**, not noise.  
- Rule of thumb: each log record / span / metric series should be **useful**. Formatting and writing is not free — apps can spend more time on telemetry than real work if you over-log.

---

## Don’t couple message and destination

**Smell:** `println!` / `eprintln!` / ad-hoc file writes as the production “logger”.

**Why:** production wants different destinations (stdout, stderr, file, socket, collector). `println!` ties **what** you log to **where** it goes.

**Prefer:** a logging/tracing facade + configurable subscriber/exporter (e.g. `tracing` + layers).

---

## Structured data so you can query later

Free-text lines are hard to turn into “how many failures last hour by error type?”

**Prefer:** structured fields (often JSON at the edge) so tools can filter and group.

Events alone have **no duration** and **no hierarchy**. Real work is hierarchical — that is what **spans** are for.

---

## Units of work → spans

A good rule of thumb: view app logic as a series of **units of work**.  
Each unit has a **start** and an **end**, and may contain **sub-units** (a tree, not a flat list).

**Span** = that unit in telemetry: start/end, optional parent, structured fields.

### What to record for each unit

| Field | Meaning |
|-------|---------|
| **Duration** | How long it took |
| **Outcome** | Success or failure (and often error class) |

Without duration and outcome, a “span” is little better than a log line with extra cost.

### What counts as a unit of work?

Use a span when the work is a **meaningful chunk** relative to the bigger operation (e.g. handling one request, one job, one pipeline step) — something you might care about when debugging **slowness** or **failure**.

**Usually yes (good span candidates):**

| Kind of work | Why |
|--------------|-----|
| Handle one **incoming request** / message / job | Outer unit; parent for everything inside |
| **I/O that can take time or vary** — DB query, HTTP client call, read large body, object storage, queue publish | Duration and failure matter |
| A **pipeline step** that is a real stage of the domain (parse document, plan write, apply migration) | Nested under the outer unit |
| Work that can **fail in a way operators care about** | Outcome on the span helps queries |
| Background task you **spawn** and still care about as related work | Own span; link to parent (child or follows) |

**Usually no (not worth a span by itself):**

| Kind of work | Why |
|--------------|-----|
| Parse one header / one tiny field | Very fast; duration barely varies |
| Simple field access, pure map/filter of small data | No useful duration story |
| Every private helper on the happy path | Noise; hides real slow steps |
| Pure thinking code with no I/O and microsecond work | Prefer tests + maybe a log on rare error |

### How fine-grained?

Ask: **Could this step take a meaningful slice of the parent’s time, or fail in a way we’d query later?**

- **Relative to the parent.** Parsing a request body is a good unit under “handle request”; parsing a single header usually is not.  
- **Variable duration.** If duration is always ~constant and tiny, a span rarely helps. If it grows with input size, concurrency, or network, span it.  
- **Not every function.** Prefer spans at **boundaries of meaningful work**, not at every `fn`.  
- **Nest, don’t flatten.** Outer request span → DB span → optional smaller steps that still matter.  
- **When unsure:** one span on the outer unit + spans on **external I/O** first; add more only when debugging shows a missing middle.

### Web server example (from Zero to Production / flashcards)

Outer unit of work: **handle this HTTP request** (one span for the whole request is normal).

Inside that request:

| Candidate | Span? | Why |
|-----------|--------|-----|
| Parse **one header** value | **No** | Very fast; duration barely varies |
| Parse / stream the **request body** | **Yes** | Can take real time; varies with size and how the client sends it |
| Call the **database** | **Yes** | I/O; variable; can fail |
| Call a **downstream HTTP API** | **Yes** | Same |
| Format a simple response string | **No** | Cheap and stable (unless huge / expensive) |

**Same idea in plain words:** header parsing is not a unit of work worth its own span. Body parsing is — it is a meaningful piece of work that can dominate request time when the payload is large or slow.

```text
request  (span: handle_request)
  ├── (no span) read Content-Type header
  ├── body     (span: parse_body)     ← unit of work
  ├── db       (span: load_user)      ← unit of work
  └── (no span) write small JSON fields
```

### Events vs units of work

- **Event / log:** “this happened” at a point in time (no duration).  
- **Unit of work / span:** “this whole chunk ran from here to there.”  

Do not turn every info log into a span. Do not replace useful events inside a span with silence — nest events **inside** the unit they belong to.

---

## Logs vs spans

| | Log / event | Span |
|--|-------------|------|
| Duration | No | Yes |
| Nesting | No parent/child | Parent/child tree |
| Use | Point-in-time facts | Work that takes time |

Use both: events for “what happened”; spans for “this whole operation and its substeps.”

---

## `tracing` field rules

Fields you later `.record(...)` must be **declared when the span is created** (e.g. `foo = tracing::field::Empty`).  
Recording an undeclared field is a **silent no-op** (no compile error) — by design for low overhead.

```rust
// Bad: record does nothing at runtime
let span = tracing::info_span!("task");
span.record("foo", 43);

// Good: field known at create
let span = tracing::info_span!("task", foo = tracing::field::Empty);
span.record("foo", 43);
```

Record **outcome** on the span for meaningful units of work (`success` / `failure`), not only log a line and leave the span empty of result.

---

## Entering a span (RAII)

Creating a span is not enough. **Enter** it so events attach to it (folder metaphor: create folder → walk into it → files land inside).

```rust
let span = tracing::info_span!("db_query", user_id = 42);
let _guard = span.enter();
tracing::info!("fetching orders");
// drop(_guard) exits the span
```

- **Create** once; **enter** many times (via guards).  
- **Exit** when the enter-guard drops.  
- **Close** when no handles left (subscriber bookkeeping).  
- **Current span:** `Span::current()` — driven by **thread-local** state in sync code.

---

## `#[instrument]`

Creates and enters a span for each call.

Useful options:

- Custom name: `#[instrument(name = "process total price")]`  
- `skip` / `skip_all` — don’t put large/sensitive args on the span by default  
- `fields(...)` — declare empty fields to fill later (`outcome`, etc.)

```rust
#[instrument(name = "process total price", skip_all, fields(outcome))]
pub fn get_total(...) -> Result<u64, anyhow::Error> {
    // on error paths:
    // Span::current().record("outcome", "failure");
    // on success:
    // Span::current().record("outcome", "success");
}
```

---

## Threads and async

### New threads

Current span is **thread-local**. A spawned thread does **not** inherit the parent span automatically.

Options:

1. Leave unlinked (if work is really unrelated).  
2. Explicit **child** if parent waits until child finishes.  
3. **Follows** relation if parent may end before background work (background tasks).

### Async / work-stealing

Futures can pause and resume on **another thread**. Sync enter-guards do **not** stick across `.await` the way you might hope.

**Instrumented future:** attach a span so each `poll` enters/exits the span.

```rust
tokio::spawn(do_work().instrument(tracing::info_span!("my_async_span")));
```

`#[instrument]` on `async fn` generates the glue (uses `.instrument` under the hood).

**Spawned tasks:** `tokio::spawn` does **not** inherit the parent span. Pass the span explicitly:

```rust
let span = tracing::info_span!("parent");
tokio::spawn(async { ... }.instrument(span));
```

---

## Filters (logs/spans)

A **filter** decides whether a record is emitted. Common axes:

- **Level** (importance)  
- **Target / module** (where it came from)

Tune so production isn’t flooded and debug can zoom in.

---

## Export: OpenTelemetry

Terminal JSON is a start; production usually needs a **collector/backend**.

Prefer **vendor-neutral** export (OpenTelemetry) over one vendor’s only SDK, when the org supports it.

In `tracing`: **Registry** holds span state/relationships; **Layer**s react (log format, OTel export, …) like middleware.

---

## Errors and panics

| Kind | Typical tool | Telemetry note |
|------|----------------|----------------|
| Recoverable | `Result` + error types / `anyhow` / report types | Span `outcome=failure`; structured error fields |
| Panic | Unwind + **panic hook** | Default prints stderr; production should feed the **same pipeline** as other telemetry |

Install a custom panic hook when panics must land in your logs/traces, not only the terminal.

```rust
std::panic::set_hook(Box::new(|panic_info| { /* emit via tracing / pipeline */ }));
```

Panic when you truly cannot continue safely, or when a hard invariant is broken — not as normal control flow.

---

## Metrics

- **Recorder** = backend/sink for metrics (`metrics` crate + exporter).  
- **describe_*** once per metric name (unit, description) for richer backends.  
- **Labels** = dimensions for filtering (method, status).  
- **Cardinality:** only **bounded, well-known** label values.  
  - Bad: `user_id`, raw URLs, unbounded free text.  
  - Series count multiplies across labels — can explode memory and cost.  
- `counter!` typically **upserts** (create if missing, then increment).  
- **Push** export: app sends on a schedule. **Pull** (e.g. Prometheus scrape): app exposes an endpoint.

Metric name = what you measure. Labels = how you slice. Prefer one metric + labels over a metric name per combination.

---

## Review checklist (diff / PR)

When new or changed production paths are in scope:

1. **Useful signals?** No firehose of debug noise; no empty “success” with no context.  
2. **Not `println!` as production logging?**  
3. **Units of work** chosen well — spans on meaningful/variable work, not every helper; duration + outcome?  
4. **Span fields** declared before `record`? Outcome recorded?  
5. **Async/spawn:** spans attached with `.instrument` / `#[instrument]`; no silent orphan spans?  
6. **Structured fields** for anything you will query later?  
7. **Metrics labels** bounded cardinality?  
8. **Panics** go through the telemetry pipeline if process is long-running?  
9. **Secrets** never in logs/span fields (also hard-rule territory).  
10. **Cost:** logging on the hot path proportional to value?

### Flags (examples)

| Smell | Prefer |
|-------|--------|
| `println!` / bare `log` without structure in server code | `tracing` + fields + subscriber |
| Span created never entered | enter / `#[instrument]` / `.instrument` |
| `record("x", …)` without declaring `x` | declare field at span create |
| `#[instrument]` on huge arg dumps / secrets | `skip` / `skip_all` + selective fields |
| `tokio::spawn` without span link when parent matters | `.instrument(span)` |
| Metric label = user id / request id unbounded | drop label or bucket/hash offline |
| Log every tiny step at info | coarser spans + targeted events |
| Span on every tiny helper / one-liner | only meaningful units of work |
| Heavy I/O or variable step with no span under a parent | add a nested unit of work |
| Panic only on stderr in a service | panic hook → same pipeline |

### Not always a smell

- No spans on pure, tiny helpers (they are not units of work).  
- Unlinked background work that is intentionally separate.  
- Verbose debug logs behind a filter, off by default in prod.  

---

## Related pack skills

| Skill | Overlap |
|-------|---------|
| `skeptic-testability` | Thinking vs shell; tests for decisions |
| `skeptic-hard-rules` | Secrets in errors/logs |
| `unit-tests` | Test quality, not production telemetry |
| `pr-description` | Honest Testing section may mention how you validated in env |
