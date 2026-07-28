---
name: observability
description: >
  Write and review production observability: structured logs, spans/traces,
  metrics, panic hooks (Rust-first: tracing, OpenTelemetry, metrics). Use when
  adding telemetry, reviewing logs/metrics/spans, or when skeptic observability
  stage runs. Not for unit-test craft alone.
---

# Observability

Make production systems **askable**: useful logs, spans, and metrics — without coupling to one sink, without silent span bugs, without unbounded metric labels.

**Full rules:** load `references/guide.md` for non-trivial work.  
**Sources:** Zero to Production in Rust (telemetry chapter); Rust Telemetry flashcards (`tracing`, metrics, panics).

## First rules (always)

1. **Observability** — answer new questions without shipping a new probe for each question.  
2. **Each signal should pay for itself** — formatting and I/O cost real time.  
3. **Separate what from where** — don’t use `println!` as production logging.  
4. **Structure fields** so you can filter/group later (not only free text).  
5. **Units of work → spans** — only for **meaningful** chunks (noticeable or variable time, failures you care about, real pipeline steps / I/O). Not every helper. Duration + outcome. See guide § Units of work.  
6. **Declare span fields before `record`** — undeclared fields are silent no-ops.  
7. **Enter spans** (or use `#[instrument]` / `.instrument`) so events nest correctly.  
8. **Async/spawn** — attach spans explicitly; thread-local does not follow tasks/threads alone.  
9. **Metrics labels** — only **bounded** dimensions; never raw user ids as labels.  
10. **Panics in services** — custom panic hook into the same pipeline as other telemetry.  

## Do (write)

- Prefer `tracing` (events + spans) over ad-hoc prints.  
- Span **units of work** that matter (request, DB, external call, heavy parse) — not micro-helpers; record `outcome` when useful.  
- `skip` / `skip_all` large or sensitive args; add only needed fields.  
- On `tokio::spawn` / new threads: pass parent link (child or follows) when work is related.  
- Export via configurable layers (JSON, OpenTelemetry, …).  
- Metrics: name + bounded labels; `describe_*` once; watch cardinality.  

## Do (review)

Flag with path:line (load guide for depth):

| Smell | Prefer |
|-------|--------|
| `println!` / unstructured print as prod log | tracing + subscriber |
| Span never entered | enter / instrument |
| `record` without field at create | declare `Empty` (or value) at create |
| Spawned task loses parent span | `.instrument(span)` |
| Metric label with unbounded values | drop or bound the dimension |
| Hot path logs everything at info | coarser spans + filters |
| Span on every tiny helper | span only meaningful units of work |
| Missing span on heavy I/O / variable steps | add unit-of-work span |
| Panic only on stderr in long-running service | panic hook → pipeline |
| Secrets in log/span fields | redact / never attach |

When in scope: `observability: ok` (one line) or findings.

## Scope

- **In:** logs, spans, metrics, panic hooks, export, async/thread span attachment, label cardinality.  
- **Out:** pure unit-test style (→ `unit-tests`); full multi-lens review (→ `skeptic`); product feature design alone.  

## Related

- Skeptic stage: `skeptic-observability` loads this.  
- Hard rules: secrets in errors/logs.  
