# `select!`, concurrency inside a task, and spawning

Reference for reviewing how async code runs several things at once. Tokio-first.

---

## 1. The model

There are two kinds of concurrency in async Rust:

| | **Inside one task** (`select!`, `join!`, merge/race, `FuturesUnordered`) | **Across tasks** (`tokio::spawn`, `JoinSet`) |
|--|--|--|
| Parallel on several cores | No — one task, one thread at a time | Yes |
| Can borrow local data | Yes | No — needs `'static` (owned data, `Arc`) |
| Cost | No extra allocation | One allocation per task |
| Number of operations | Fixed in code (`select!`, `join!`), or a dynamic set (`FuturesUnordered`) | Any number |
| When the parent is cancelled | Children are cancelled with it | Children keep running (handle drop detaches) |
| Visible to the runtime / tokio-console | As one task | Each is its own task |

**Pick inside-one-task** for a fixed set of IO-bound operations that should stop when the parent stops (e.g. one task per connection that `select!`s over reads, writes, and messages). **Spawn** for CPU-heavy or parallel work, a variable number of jobs, work that must finish even if the caller goes away, and futures that would otherwise risk futurelock.

---

## 2. `select!` rules

`tokio::select!` waits on several branches and returns when the first completes, **dropping (cancelling) the others**.

### 2.1 Every branch future must be cancel safe — or survive between iterations

```rust
// Worse: a fresh sleep every iteration never fires while messages keep arriving
loop {
    tokio::select! {
        Some(msg) = rx.recv() => handle(msg),
        _ = sleep(Duration::from_secs(1)) => break,
    }
}

// Better: create the deadline once, pin it, poll it through &mut
let deadline = sleep(Duration::from_secs(1));
tokio::pin!(deadline);
loop {
    tokio::select! {
        Some(msg) = rx.recv() => handle(msg),
        _ = &mut deadline => break,
    }
}
```

Use `tokio::pin!` / `std::pin::pin!` for a future that lives across the loop; `Box::pin` when you replace it inside the loop (`fut.set(...)` or reassignment).

### 2.2 Do not poll a finished future

A pinned future polled again after it completed panics ("`async fn` resumed after completion"). Guard it:

```rust
let mut done = false;
loop {
    tokio::select! {
        res = &mut op, if !done => { done = true; handle(res); }
        Some(msg) = rx.recv() => { if restart(&msg) { op.set(action(msg)); done = false; } }
        else => break,
    }
}
```

### 2.3 Preconditions and `else`

- A precondition (`, if cond`) is checked once per `select!` call. Do not use racy ones (`if !sleep.is_elapsed()`); let the future's own completion decide.
- A branch is also disabled when its pattern does not match (`Some(msg) = rx.recv()` on a closed channel).
- If every branch can become disabled, add `else`, or `select!` panics.

### 2.4 Fairness and `biased;`

By default branches are polled in random order so one busy branch cannot starve the rest. `biased;` polls top to bottom: then **you** own fairness — put shutdown first so a busy stream cannot hide it, and make sure a hot branch cannot starve the others.

```rust
loop {
    tokio::select! {
        biased;
        _ = shutdown.cancelled() => break,
        Some(msg) = stream.next() => handle(msg),
    }
}
```

### 2.5 No blocking inside branches

All branches run on one task; blocking in one stalls all. For parallel work, spawn each and `select!` over the `JoinHandle`s.

### 2.6 Prefer a simpler tool when it fits

| Need | Prefer over a `select!` loop |
|------|-----------------------------|
| Handle every item from several streams | Merge the streams (`StreamExt::merge`, `tokio_stream::StreamMap`) |
| First result wins, discard the rest | `timeout`, race helpers |
| Wait for all, propagate errors after | `join!` then check results |
| Many independent jobs | `JoinSet` |

### 2.7 Batch instead of one event per wakeup

`loop { select! { … } }` handles one event per iteration. After a slow step, many events are often waiting. Pull everything ready, then prioritize, drop redundant events, and coalesce:

```rust
while let Some(first) = rx.recv().await {
    let mut batch = vec![first];
    while let Ok(event) = rx.try_recv() { batch.push(event); }
    for event in coalesce(batch) { process(event).await; }
}
```

---

## 3. `FuturesUnordered` and buffered streams

They run many futures inside one task. Two hazards:

- **Futurelock:** while the loop body awaits other work, futures in the set are not polled; if one of them holds a lock or permit the body needs, the task hangs (see `cancellation.md` § 3.5). Prefer `JoinSet`, or push the extra work into the set.
- **Hidden ordering points:** a buffered stream (`buffer_unordered`) stops starting new work while the consumer is busy. A producer that holds a lock across a yield, plus a consumer that takes the same lock, deadlocks. Buffer data (channels between tasks), not code.

---

## 4. Spawning

- **Handle the result.** `JoinHandle` resolves to `Result<T, JoinError>`; `Err` means the task panicked or was cancelled. Dropping the handle detaches the task and hides its panic.
- **`Send` + `'static`** — see `blocking.md` § 5. The `!Send` error appears at `tokio::spawn`, far from its cause.
- **Name the owner.** Every spawned task should have an owner that can stop it and wait for it (`JoinSet`, `TaskTracker`) — see `lifetimes-and-load.md`.

---

## Sources

- https://tokio.rs/tokio/tutorial/select — Select (Tokio tutorial)
- https://docs.rs/tokio/latest/tokio/macro.select.html — `tokio::select!`
- https://tokio.rs/tokio/tutorial/spawning — Spawning (Tokio tutorial)
- https://blog.yoshuawuyts.com/futures-concurrency-3/ — Futures Concurrency III: select! (Yoshua Wuyts)
- https://without.boats/blog/let-futures-be-futures/ — Let futures be futures (without.boats)
- https://without.boats/blog/futures-unordered/ — FuturesUnordered and the order of futures (without.boats)
- https://without.boats/blog/the-scoped-task-trilemma/ — The scoped task trilemma (without.boats)
- https://matklad.github.io/2025/05/14/scalar-select-aniti-pattern.html — Scalar Select Anti-Pattern (matklad)
- https://rfd.shared.oxide.computer/rfd/0609 — Futurelock (Oxide)
- https://sunshowers.io/posts/nextest-and-tokio/ — How (and why) nextest uses tokio (Rain)
