---
name: skeptic-async
description: >
  Async Rust review (Tokio-first): cancellation and cancel safety, futurelock,
  blocking the runtime, shared state (std vs tokio Mutex, actors, channels),
  select! and spawn, task lifetimes and graceful shutdown, backpressure and
  queue sizing (Little's Law), and async I/O performance. Use when the user asks for an async Rust
  review, or when skeptic runs and the diff contains async code.
---

# Skeptic async

**Question:** Is this async code correct when futures are cancelled, does it keep the runtime responsive, does every task have an owner that can stop and wait for it, and does it hold up under load?

## Load

Always, when the diff contains `async`, `.await`, `tokio::`, futures, or a hand-written `Future`:

1. `references/cancellation.md` — cancel safety vs correctness, where cancellation comes from, futurelock, cancel-safe patterns and API design  
2. `references/blocking.md` — time budget between awaits, which tool for which work, `block_on` in async, bridging, `Send`/`'static`  
3. `references/shared-state.md` — std vs tokio Mutex, guards across `.await`, actors, channels, bounded queues  
4. `references/select-and-tasks.md` — `select!` rules, concurrency inside one task vs spawning, `FuturesUnordered` hazards, batching  
5. `references/lifetimes-and-load.md` — structured concurrency, `JoinHandle`/`JoinSet`/`TaskTracker`/`CancellationToken`, graceful shutdown, backpressure, sizing queues and caps with Little's Law, timeouts  
6. `references/performance.md` — when the diff is on a hot path or does I/O: per-core work, batching, buffers, completion-based I/O, measurement  

## Do

1. **Find every cancellation point** that reaches changed code: `select!`, `timeout`, `try_join*`, `abort()`, runtime shutdown, client disconnect, cancelled parents. For each future that can be dropped there: is it cancel safe? If not, is it resumed (`&mut` + pin), split (reserve → sync commit), progress kept outside it, or moved into its own task?  
2. **Invariants across `.await`:** flag state left broken across an await (`take()`, half-updated fields), especially under `tokio::sync::Mutex`. Flag state read before an await and trusted after it.  
3. **Futurelock:** flag `&mut fut` in `select!` with an await in a handler or after the `select!` while `fut` is alive; flag `FuturesUnordered` / buffered-stream loops that await other work in the body.  
4. **Blocking:** flag `std::thread::sleep`, sync I/O, blocking channels, long CPU work, `par_iter` in async code, blocking in `Drop`, and `block_on` inside the runtime. Check the right tool: inline / `spawn_blocking` / rayon + oneshot / dedicated thread / `block_in_place`.  
5. **Shared state:** flag std guards alive at an `.await` (an explicit `drop` is not enough), `tokio::sync::Mutex` around plain data, unbounded channels, blocking channels, actor cycles that prevent shutdown, and cycles of bounded channels.  
6. **`select!`:** non-cancel-safe branches in loops, recreated timers, polling a finished future, racy preconditions, missing `else`, `biased;` without a fair order, blocking inside a branch. Prefer merge / race / `join!` / `JoinSet` when they fit.  
7. **Task ownership:** flag detached `tokio::spawn`, ignored `JoinHandle` results, `abort()` as normal control flow, missing shutdown signal, shutdown that does not wait, sender/worker drop-order deadlocks, async cleanup only on the cancel path.  
8. **Load:** every queue and fan-out bounded on purpose; backpressure reaches the producer; per-call delays serialized when producers are concurrent; limits have a ceiling and a failure threshold; every remote call and retry has a timeout.  
9. **Sizing (Little's Law, L = λ × W):** for every new or changed channel capacity, concurrency cap, or pool size, ask for peak arrival rate (λ) and per-item time (W) if the code does not state them. Check the consumer keeps up on average (otherwise no capacity helps), capacity ≥ peak rate × longest stall to absorb, caps ≥ λ × W, and state worst-case memory (capacity × item size) and added wait (capacity / consumer rate). Flag magic numbers and "big to be safe" capacities.  
10. **Performance (hot paths):** apply `performance.md` — keep work and data local, batch I/O and messages, avoid per-item allocation and shared atomics on the hot path, keep buffers alive until I/O completes, measure before adding complexity.  
11. **Lints:** suggest enabling clippy `await_holding_lock`, `await_holding_refcell_ref`, `await_holding_invalid_type`, `large_futures`, `future_not_send`, `let_underscore_future`, `unused_async`, `async_yields_async` when the crate does not already; tokio-console for hung tasks.  
12. **Findings:** skeptic concern format, with the hazard named (cancellation, futurelock, blocking, detached task, unbounded queue, unsized capacity, …), path:line, what goes wrong at runtime, and the smallest fix. LETTER options for real design forks (e.g. actor vs std mutex, spawn vs intra-task).  
13. When async code is in scope: `async: ok` (one line naming the main paths checked) or findings. No async code → `none` with one-line why.  

## Do not

- Timeouts as a pass/fail ban (→ hard-rules `HR-io-timeout`); report the cancel-safety side here  
- Tracing spans on spawned tasks (→ observability)  
- Component API shape and performance budgets (→ architecture); report runtime behavior here  
- Ask for a runtime switch (e.g. thread-per-core) without a measured reason  
- Flag cancel-unsafe futures that can never be cancelled in context (owned by a task that is never aborted) — say why it is safe instead  

## Output section title

`## Async` (standalone) — or the extra `## Async` section in a skeptic report.
