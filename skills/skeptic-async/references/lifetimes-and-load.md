# Task lifetimes, shutdown, and load

Reference for reviewing who owns spawned work, how it stops, and how the system behaves under load. Tokio-first.

---

## 1. The model: structured concurrency

Plain function calls form a tree: when a function returns, everything it started has finished; when it fails, the error reaches its caller; when it is cancelled, its children are cancelled. **Structured concurrency** keeps those three guarantees with concurrent work:

1. **Cancellation propagates** — stopping a parent stops its children.
2. **Errors propagate** — a child's failure reaches someone who can handle it.
3. **Ordering holds** — when a parent returns, its work is done.

Futures composed with `.await`, `join!`, `select!` keep these guarantees. **`tokio::spawn` breaks them**: dropping a `JoinHandle` detaches the task. A detached task has no parent: nobody can cancel it, wait for it, or see its panic. Global "background tasks" are structured only if their pool is reachable — owned, cancellable, and joined.

---

## 2. Failure modes

### 2.1 Fire-and-forget spawn

```rust
// Worse: errors unwrap into a panic nobody sees; shutdown cannot stop or wait for it
async fn handler(body: String) -> StatusCode {
    tokio::spawn(async move { send_email(body).await.unwrap(); });
    StatusCode::ACCEPTED
}

// Better: an owned worker (actor) with a bounded queue; main runs it and waits for it
async fn handler(jobs: &mpsc::Sender<String>, body: String) -> StatusCode {
    match jobs.try_send(body) {
        Ok(()) => StatusCode::ACCEPTED,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE, // queue full: push back
    }
}
```

### 2.2 Ignored `JoinHandle` results

```rust
// Worse: a panic in any task is silently dropped
for job in jobs { tokio::spawn(run(job)); }

// Better: own the tasks and look at every result
let mut set = tokio::task::JoinSet::new();
for job in jobs { set.spawn(run(job)); }
while let Some(res) = set.join_next().await {
    match res {
        Ok(Ok(())) => {}
        Ok(Err(e)) => tracing::warn!(error = %e, "job failed"),
        Err(e) if e.is_panic() => std::panic::resume_unwind(e.into_panic()),
        Err(e) => tracing::warn!(error = %e, "job cancelled"),
    }
}
```

### 2.3 One child fails, siblings run on

Waiting for every sibling before reporting the first error can delay it without limit. On the first failure, cancel the rest (shared `CancellationToken`, `abort_all`), then drain.

### 2.4 No shutdown signal, or shutdown that does not wait

A task with no way to be told to stop runs until the runtime kills it at a random await. A `main` that returns without waiting skips flushes and loses work.

### 2.5 Shutdown deadlock on drop order

A worker that exits when its channel closes hangs if its `Sender` is still alive when you wait for it — easy to miss on the error path. Drop every sender before awaiting the worker, and make the happy and error paths shut down the same way.

### 2.6 Async cleanup in `Drop`

`Drop` is synchronous; there is no async drop. A destructor cannot await a flush or a graceful close, and blocking in it stalls the runtime. Give the type an explicit `async fn close(self)` / `shutdown(self)`, call it on every exit path, and make sure cleanup runs on normal completion too — not only on the cancel branch (cancel-only paths are the least tested). Correctness must not depend on cleanup running: values can leak and futures can be dropped without it.

---

## 3. Graceful shutdown with Tokio

Three steps: **detect** → **notify** → **wait**.

```rust
use tokio_util::{sync::CancellationToken, task::TaskTracker};

#[tokio::main]
async fn main() {
    let token = CancellationToken::new();
    let tracker = TaskTracker::new();

    for id in 0..4 {
        let token = token.child_token();
        tracker.spawn(async move {
            loop {
                tokio::select! {
                    _ = token.cancelled() => { flush(id).await; break; } // stop at a consistent point
                    job = next_job(id) => process(job).await,
                }
            }
        });
    }

    // detect: Ctrl-C (shut down even if registering the handler fails)
    let _ = tokio::signal::ctrl_c().await;

    token.cancel();        // notify
    tracker.close();       // no more tasks expected
    tracker.wait().await;  // wait until every task has exited
}
```

| Tool | What it does | Watch for |
|------|-------------|-----------|
| `CancellationToken` | Cooperative stop signal; `cancelled().await` is cancel safe | Clones cancel each other; `child_token()` lets a subtree be cancelled alone; `drop_guard()` cancels on scope exit (including panic and `?`) |
| `TaskTracker` | Waits for tasks; frees each task's memory when it exits | `wait()` returns only after `close()`; dropping it neither aborts nor waits; does not report panics |
| `JoinSet` | Owns tasks and their results | Dropping it **aborts** every task; `abort_all` still needs a `join_next` drain; `shutdown()` = abort + drain (ignores panics); `join_all` panics on the first `JoinError`; results pile up until drained |
| `JoinHandle` | One task | Drop = detach; `abort()` stops at the next await (avoid as normal control flow) |

Shutdown multiple triggers through one place (e.g. `select!` over `ctrl_c()` and an internal channel).

---

## 4. Load: bounds, backpressure, timeouts

### 4.1 Every queue needs intentional backpressure

When producers are faster than consumers, something must slow them down or reject them. Otherwise memory grows until the process fails.

- **Accidental backpressure is fragile.** A slow step can be the only thing holding producers back; optimize it and the pile-up appears elsewhere.
- **Bound by the resource that runs out.** Count items *and* bytes when item sizes vary; apply the larger delay or limit.
- **Apply the push-back where the producer waits.** Calls that already wait for completion (reads, flushes) get it for free; fire-and-forget calls need it added.
- **Concurrent producers defeat per-call delays.** If each of N callers sleeps on its own, all N still pile on — serialize the delay or apply it to the stream as a whole.
- **Every limit needs a ceiling and a failure threshold.** An uncapped delay can hide a dead dependency indefinitely; pair the delay with a point where the component reports itself faulted.
- **Rate limits must feed backpressure.** A throttle in front of an unbounded buffer just moves the pile-up into the buffer.
- **Fast acknowledgement hides latency.** If writes are acknowledged before completion, measure the latency of the next operation that must wait for them.

### 4.2 Size every bound with Little's Law

A bound needs a number, and the number needs a reason. **Little's Law** gives it: the average number of items inside a system (**L**) equals the average rate they arrive (**λ**) times the average time each one stays (**W**).

```text
L = λ × W        items in the stage = arrivals per second × seconds each item stays
```

It holds for any stable part of the system on its own: one channel, one worker pool, the permits of one `Semaphore`, the in-flight calls to one dependency. When a diff adds or changes a capacity, ask for λ and W if the code does not state them, then check:

1. **Is the stage stable?** The consumer's service rate is `workers / time per item`. If the average arrival rate is at or above it, the queue fills no matter how large it is; capacity only decides when push-back starts. The fix is more consumers, batching, or shedding load — not a bigger channel.
2. **What is the buffer for?** A queue rides out bursts and consumer stalls. Capacity ≥ peak arrival rate × longest stall it must absorb, plus headroom. Use peak rate and high-percentile time, not averages: the law speaks of long-term means, which hide spikes and daily cycles.
3. **What does it cost?** Worst-case memory = capacity × item size (count bytes when sizes vary). Worst-case added wait = capacity / consumer rate. A queue longer than the latency budget holds work that is stale by the time it runs.
4. **Is the concurrency cap big enough?** Permits or workers needed = λ × W. At 2 000 req/s and 50 ms per call, 100 calls are in flight on average; a cap of 16 limits throughput to 16 / 0.05 s = 320 req/s.
5. **Does W feed back into λ?** Contention makes W grow as L grows, and timeouts with retries make λ grow as W grows. Together they turn a slowdown into a collapse. Retries must be bounded and count against the same limit as first attempts.
6. **What do production numbers say?** Read the law backwards from metrics: queue depth / throughput = time spent waiting. When a full queue rejects work, use the accepted rate, not the offered rate.

```rust
// Worse: where does 10_000 come from? How much memory does it hold, how long does it take to drain?
let (tx, rx) = mpsc::channel(10_000);

// Better: the number follows from stated load.
// Peak 5 000 msg/s; the writer can stall 200 ms on flush → 5 000 × 0.2 s = 1 000 queued; 2× headroom.
// ~4 KiB per message → ~8 MiB worst case; the writer drains 10 000 msg/s → a full queue adds ≤ 0.2 s.
const WRITE_QUEUE_CAPACITY: usize = 2_000;
let (tx, rx) = mpsc::channel(WRITE_QUEUE_CAPACITY);
```

Flag capacities and caps (`channel(n)`, `Semaphore::new(n)`, `buffer_unordered(n)`, pool sizes, byte budgets) that have no stated rate, time, and memory cost — and capacities that are large "to be safe", which only delay push-back and hide an unstable stage.

### 4.3 Bound concurrency explicitly

Async tasks are cheap, so nothing stops a burst from creating a million of them. Add explicit caps — a `Semaphore` around a resource, a bounded `JoinSet`, `buffer_unordered(n)` — overall and per dependency. Do not rely on running out of memory as the limit.

### 4.4 Timeouts on remote calls

Every network or RPC call needs a timeout (`tokio::time::timeout`), including each retry attempt. Remember that a timeout is a cancellation: the operation it wraps must be cancel safe (`cancellation.md`).

### 4.5 Debuggability

Async stacks show little about what an idle task is waiting for, and channels lose the sender's context. Instrument spawned tasks with tracing spans, carry IDs across message hops, and use tokio-console to see stuck tasks.

---

## Sources

- https://tokio.rs/tokio/topics/shutdown — Graceful Shutdown (Tokio)
- https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html — `CancellationToken`
- https://docs.rs/tokio-util/latest/tokio_util/task/task_tracker/struct.TaskTracker.html — `TaskTracker`
- https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html — `JoinSet`
- https://blog.yoshuawuyts.com/tree-structured-concurrency/ — Tree-Structured Concurrency (Yoshua Wuyts)
- https://blog.yoshuawuyts.com/replacing-tasks-with-actors/ — Replacing Background Tasks With Actors (Yoshua Wuyts)
- https://blog.yoshuawuyts.com/tasks-are-the-wrong-abstraction/ — Tasks are the wrong abstraction (Yoshua Wuyts)
- https://matklad.github.io/2019/08/23/join-your-threads.html — Join Your Threads (matklad)
- https://matklad.github.io/2018/07/24/exceptions-in-structured-concurrency.html — Exceptions vs Structured Concurrency (matklad)
- https://matklad.github.io/2023/10/11/unix-structured-concurrency.html — UNIX Structured Concurrency (matklad)
- https://without.boats/blog/asynchronous-clean-up/ — Asynchronous clean-up (without.boats)
- https://rfd.shared.oxide.computer/rfd/0445 — Crucible Upstairs Backpressure (Oxide)
- https://rfd.shared.oxide.computer/rfd/0079 — Rust approaches to concurrency in the control plane (Oxide)
- https://brooker.co.za/blog/2018/06/20/littles-law.html — Telling Stories About Little's Law (Marc Brooker)
- https://en.wikipedia.org/wiki/Little%27s_law — Little's law (Wikipedia)
