---
name: skeptic-async
description: >
  Async Rust review (Tokio-first): cancellation and cancel safety, cancel-safe
  API design and cancel tests, stop signals, futurelock,
  blocking the runtime, shared state (std vs tokio Mutex, channels), sans-IO
  actors, select! and spawn, task lifetimes and graceful shutdown, backpressure
  and queue sizing (Little's Law), and async I/O performance. Use when the user
  asks for an async Rust review, or when skeptic runs and the diff contains
  async code.
---

# Skeptic async

**Question:** Is this async code correct when futures are cancelled, does it keep the runtime responsive, does every task have an owner that can stop and wait for it, and does it hold up under load?

## Load

Always, when the diff contains `async`, `.await`, `tokio::`, futures, or a hand-written `Future`:

1. `references/cancellation.md` — the model (three kinds of stopping, four review viewpoints, how far damage reaches), where cancellation comes from, failure modes, futurelock, Tokio cancel-safety table  
2. `references/cancel-safe-design.md` — making APIs cancel safe and marking the rest, calling cancel-unsafe code without cancelling it, tests that prove cancel safety  
3. `references/blocking.md` — time budget between awaits, which tool for which work, `block_on` in async, bridging, `Send`/`'static`  
4. `references/shared-state.md` — std vs tokio Mutex, guards across `.await`, channels, bounded queues  
5. `references/actors.md` — sans-IO actors: job struct + private state, `serve` at the edge, DAG shutdown, enqueue-then-run tests  
6. `references/select-and-tasks.md` — `select!` rules, concurrency inside one task vs spawning, `FuturesUnordered` hazards, batching  
7. `references/lifetimes-and-load.md` — structured concurrency, `JoinHandle`/`JoinSet`/`TaskTracker`/`CancellationToken`/`AbortOnDropHandle`, choosing a stop signal, graceful shutdown with a deadline, backpressure, sizing queues and caps with Little's Law, timeouts  
8. `references/performance.md` — when the diff is on a hot path or does I/O: per-core work, batching, buffers, completion-based I/O, measurement  

## Do

1. **Find every cancellation point** that reaches changed code: `select!`, `timeout`, `try_join*` / `try_for_each_concurrent`, `run_until_cancelled`, `abort()`, runtime shutdown, client disconnect, cancelled parents. Read each changed async fn from the four viewpoints (`cancellation.md` § 1): the cancelled future (what has changed at each await), the parent (does it drop or recreate it), the runtime (shutdown order), other tasks (how each is stopped). For each future that can be dropped: is it cancel safe? If not, is it resumed (`&mut` + pin), split (reserve → sync commit), progress kept outside it, merged as a stream, or moved into its own task? Rank by how far damage reaches: state outside the process is the most serious.  
2. **Invariants across `.await`:** flag state left broken across an await (`take()`, half-updated fields), especially under `tokio::sync::Mutex`. Flag state read before an await and trusted after it.  
3. **Side effects that must finish:** flag `try_join!`, `try_join_all`, `try_for_each_concurrent`, `try_collect` over flushes, stops, deletes, or state outside the process; suggest `join!` then check each result (or `cancel-safe-futures` `*_then_try` as an option). Flag request handlers that the framework drops on client disconnect while doing such work; suggest a task per request.  
4. **Marked and proven:** flag a new or changed public async method that callers will `select!` or `timeout` with no `# Cancel safety` doc section; a cancel-unsafe method named `next` / `recv` (or `reserve` / `acquire` when more than the queue place is lost); a new one-call `send`-style convenience that hides a prepare/commit split; and a cancel-unsafe function placed under `select!`, `timeout`, or `try_*` — or a method documented cancel safe — with no test that drops it at an await (`cancel-safe-design.md` § 3).  
5. **Cleanup on cancel:** flag cleanup in the `None` arm of `run_until_cancelled` / `with_cancellation_token` for state that lived inside the dropped future (`cancellation.md` § 3.11); invariants repaired only on the happy path with no drop guard; `unwrap()` on sends to sibling tasks that may already be gone at shutdown (§ 3.12).  
6. **Futurelock:** flag `&mut fut` in `select!` with an await in a handler or after the `select!` while `fut` is alive; flag `FuturesUnordered` / buffered-stream loops that await other work in the body.  
7. **Blocking:** flag `std::thread::sleep`, sync I/O, blocking channels, long CPU work, `par_iter` in async code, blocking in `Drop`, and `block_on` inside the runtime. Check the right tool: inline / `spawn_blocking` / rayon + oneshot / dedicated thread / `block_in_place`.  
8. **Shared state:** flag std guards alive at an `.await` (end the scope; an explicit `drop` may not satisfy the compiler), `tokio::sync::Mutex` around plain data or never held across an `.await`, `Arc<Mutex<T>>` passed around with `lock()` calls in async code instead of a wrapper struct that locks only in non-async methods, methods that return a guard or a reference into the lock, `dashmap` or `parking_lot` locks taken inside `async fn` (their guards can be `Send`, so the compiler will not catch an await under them), IO resources such as streams inside a locked collection, unbounded channels, and blocking channels.  
9. **Actors:** when the diff has a mailbox, a job struct that talks over a channel, or a task that owns state and receives messages, apply `actors.md`. Flag state that holds a `Receiver` / socket / `JoinHandle`, `.await` inside `step` / `handle_message`, `tokio::spawn` in the job-struct constructor (high on new or changed actors), a method that folds the oneshot, a cycle of senders or of bounded `send().await`, a heartbeat-only actor, an in-process actor framework, and an actor that could have been a sync method. Public `step` / a free `pub fn` that re-passes `Sender` / a public `Sender<Request>` that leaks the request type → architecture (`### Component API`).  
10. **`select!`:** non-cancel-safe branches in loops, recreated timers, polling a finished future, racy preconditions, missing `else`, `biased;` without a fair order, blocking inside a branch. Prefer merge / race / `join!` / `JoinSet` when they fit. Flag a **scalar select** — one event per wakeup — when handling an event has a fixed cost (a job, a flush, a write), events can replace or combine, or some inputs must win; suggest `recv_many` / `ready_chunks` and a pure `plan_batch` that prioritizes, eliminates, and coalesces (`select-and-tasks.md` § 2.7). In a batching loop check the cap on every drain, per-channel handling of a closed channel, and that each source keeps its own order. Flag a long-running task that waits between steps with `sleep(…).await` instead of a deadline in its state (§ 2.8).  
11. **Task ownership:** flag detached `tokio::spawn`, helpers that spawn and return nothing that owns the task (the call site cannot see work still running — take a `&TaskTracker` / `&mut JoinSet` from the caller), guards (`TempDir`, permits) left in the parent while a spawned task uses what they protect, ignored `JoinHandle` results, `abort()` as normal control flow, missing shutdown signal, shutdown that does not wait, sender/worker drop-order deadlocks, async cleanup only on the cancel path. Flag `tokio::spawn` inside `Drop` (cleanup with no owner; suggest an awaited `close(self)`).  
12. **Stop signals:** flag `drop(handle)` expected to stop a task (it detaches), hand-rolled `broadcast` / `oneshot` shutdown where a `CancellationToken` fits, a `watch` stop awaited with a bare `changed()`, `abort()` before any cooperative request, a request to stop with no matching wait, and a shutdown wait with no deadline or no decision for what is still running after it (`lifetimes-and-load.md` § 3).  
13. **Load:** every queue and fan-out bounded on purpose; backpressure reaches the producer; per-call delays serialized when producers are concurrent; limits have a ceiling and a failure threshold; every remote call and retry has a timeout.  
14. **Sizing (Little's Law, L = λ × W):** for every new or changed channel capacity, concurrency cap, or pool size, ask for peak arrival rate (λ) and per-item time (W) if the code does not state them. Check the consumer keeps up on average (otherwise no capacity helps), capacity ≥ peak rate × longest stall to absorb, caps ≥ λ × W, and state worst-case memory (capacity × item size) and added wait (capacity / consumer rate). Flag magic numbers and "big to be safe" capacities.  
15. **Performance (hot paths):** apply `performance.md` — keep work and data local, batch I/O and messages, avoid per-item allocation and shared atomics on the hot path, keep buffers alive until I/O completes, measure before adding complexity.  
16. **Lints:** suggest enabling clippy `await_holding_lock`, `await_holding_refcell_ref`, `await_holding_invalid_type`, `large_futures`, `future_not_send`, `let_underscore_future`, `unused_async`, `async_yields_async` when the crate does not already; tokio-console for hung tasks.  
17. **Findings:** skeptic concern format, with the hazard named (cancellation, side effect cancelled, unmarked cancel-unsafe API, missing cancel test, cleanup after drop, stop signal, futurelock, blocking, detached task, unbounded queue, unsized capacity, actor owns IO, …), path:line, what goes wrong at runtime, and the smallest fix. LETTER options for real design forks (e.g. actor vs std mutex, spawn vs intra-task).  
18. When async code is in scope: `async: ok` (one line naming the main paths checked) or findings. No async code → `none` with one-line why.  

## Do not

- Timeouts as a pass/fail ban (→ hard-rules `HR-io-timeout`); report the cancel-safety side here  
- Tracing spans on spawned tasks (→ observability)  
- Component API shape and performance budgets (→ architecture); report runtime behavior here  
- Ask for a runtime switch (e.g. thread-per-core) without a measured reason  
- Flag cancel-unsafe futures that can never be cancelled in context (owned by a task that is never aborted) — say why it is safe instead  
- Flag lost work on a path cancelled only when the process is exiting and the loss is acceptable (Ctrl-C on a download) — say so instead  
- Require the `cancel-safe-futures` dependency — offer it next to the plain `join!`-then-check form  
- Demand a `poll_transmit` / `poll_timeout` driver on a worker whose job is serializing one resource — flag `.await` in `step` instead (`actors.md`)  
- Demand a rewrite of an existing `Handle::new()` the diff did not change  

## Output section title

`## Async` (standalone) — or the extra `## Async` section in a skeptic report.
