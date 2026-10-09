# Cancellation

Reference for reviewing what happens when a future stops being polled. Tokio-first; the model applies to any runtime.

---

## 1. The model

**A future is cancelled by dropping it** (or by never polling it again). There is no signal and no unwinding: the future's state is dropped at the `.await` where it was paused, and its `Drop` impls run. Drop is synchronous, so cancellation can only run synchronous cleanup.

- **Every `.await` is a possible early return.** A future can also be dropped before its first poll, so arguments moved into it can be dropped without its body ever running.
- **Cancellation never splits a `poll`.** It happens between awaits, so code between two `.await`s runs entirely or not at all.
- **Cancellation spreads upward and downward.** Dropping a parent drops every future it owns; any future that awaits a cancelled future is itself cancelled at that point.
- **Tasks are different.** Dropping a Tokio `JoinHandle` *detaches* the task — it keeps running. A task stops only when aborted, when the runtime shuts down, or when its future finishes.

| Term | Meaning |
|------|---------|
| **Cancel safe** | A future can be dropped before completion with no effect on the rest of the system. A local property of one future. |
| **Mostly cancel safe** | Cancel safe except that you lose your place in a fair queue (e.g. `Mutex::lock`, `Semaphore::acquire`). |
| **Cancel correct** | The whole system behaves correctly given that futures inside it may be cancelled. A global property. |
| **Futurelock** | A task stops polling future A, which holds a resource that future B (in the same task) needs. The task hangs. |

A cancel-correctness bug needs all three: a cancel-unsafe future exists, it actually gets cancelled, and that cancellation breaks a property the system relies on. Fix any one of the three.

Cancellation is often a feature: dropping a waiting `acquire` gives up a place in line instead of wasting it, and Ctrl-C should be able to stop a download. Which properties matter can change.

---

## 2. Where cancellation comes from

A review must find every one of these that reaches the changed code:

| Source | What gets dropped |
|--------|-------------------|
| `tokio::select!` | Every branch that did not win |
| `tokio::time::timeout` | The inner future when the timer wins (it is a `select!` with a sleep) |
| `try_join!`, `try_join_all`, `TryStreamExt::try_for_each(_concurrent)`, `try_collect` | All remaining futures after the first `Err` |
| `JoinHandle::abort()`, `JoinSet` drop / `abort_all` | The task, at its next `.await` |
| Runtime shutdown (end of `#[tokio::main]`, tests, a dropped runtime, panic unwinding) | Every task, at its next `.await`, in arbitrary order |
| A web framework that drops the handler when the client disconnects | The request handler — cancellation triggered by a remote client |
| Any parent future being dropped | All of its children |

---

## 3. Failure modes

### 3.1 A value owned by a losing branch is lost

```rust
// Worse: when the tick wins, `value` is dropped with the send future
for value in values {
    tokio::select! {
        res = tx.send(value) => if res.is_err() { break },
        _ = interval.tick() => println!("tick"),
    }
}

// Better: wait for capacity (mostly cancel safe), then send synchronously
let mut values = values.into_iter().peekable();
while values.peek().is_some() {
    tokio::select! {
        permit = tx.reserve() => match permit {
            Ok(permit) => permit.send(values.next().unwrap()),
            Err(_) => break,
        },
        _ = interval.tick() => println!("tick"),
    }
}
```

The same bug hides inside `timeout`:

```rust
// Worse: msg is lost every time the timeout fires
let msg = next_message();
match timeout(Duration::from_secs(5), tx.send(msg)).await { … }

// Better: reserve first, build the message only once you hold a permit
match timeout(Duration::from_secs(5), tx.reserve()).await {
    Ok(Ok(permit)) => permit.send(next_message()),
    Ok(Err(_)) => return,
    Err(_) => println!("no capacity for 5s"),
}
```

**Pattern:** split an operation into an async "get ready" step that is (mostly) cancel safe and a synchronous "commit" step.

### 3.2 Partial progress is lost

```rust
// Worse: if cancelled, you cannot tell how much of `buffer` was written
writer.write_all(&buffer).await?;

// Better: a cursor records progress, so a retry resumes where it stopped
let mut cursor = std::io::Cursor::new(buffer);
while cursor.has_remaining() {
    tokio::select! {
        res = writer.write_all_buf(&mut cursor) => res?,
        _ = shutdown.cancelled() => return Ok(()),
    }
}
```

**Pattern:** keep progress outside the future — in a `&mut` argument (`bytes::Buf`) or a field on `self`.

### 3.3 Read, then more async work, in one future

```rust
// Worse: cancelled at the second await → the message is gone
async fn next(&mut self) -> Message {
    let message = self.inner.next().await;  // removed from the stream
    self.process(&message).await;           // cancel here
    message
}
```

Fixes:
- **Store progress on `self`** (`self.pending: Option<Message>`) before the next await; process it first on the next call.
- **Two-phase API:** make `recv()` do only the cancel-safe read and return a second future that must be awaited for the rest.
- **Move the rest into a task**, and store its handle on `self` *before the next await* so a cancelled call can pick it up.

### 3.4 An invariant broken across an `.await` under a Tokio mutex

```rust
// Worse: cancellation between take() and the assignment leaves None forever
let mut guard = state.lock().await;
let current = guard.take().expect("always Some");
*guard = Some(current.advance().await);   // cancel here: guard drops, lock released, state is None
```

`tokio::sync::Mutex` does not poison, so the next caller sees the broken state with no warning. In order of preference:
1. Let one task own the state and receive requests over a channel (actor).
2. Use `std::sync::Mutex` and never hold it across an `.await`.
3. If the lock must span an await: restore invariants before every await, or put the restore in a `Drop` guard, or validate and repair state each time the lock is taken — and document that the code is not cancel safe.

### 3.5 Futurelock: a future you stopped polling holds a lock

```rust
// Worse: op1 is queued for the lock, then never polled again; op2 waits behind it forever
let mut op1 = Box::pin(do_thing(lock.clone()));
tokio::select! {
    _ = &mut op1 => {}
    _ = sleep(Duration::from_millis(500)) => {
        do_thing(lock.clone()).await;   // hangs
    }
}

// Better: give op1 its own task; the runtime keeps polling it
let mut op1 = tokio::spawn(do_thing(lock.clone()));
tokio::select! {
    _ = &mut op1 => {}
    _ = sleep(Duration::from_millis(500)) => {
        do_thing(lock.clone()).await;
    }
}
```

Same trap: a `FuturesUnordered` / `FuturesOrdered` loop that awaits something else in its body while other futures in the set are mid-operation.

```rust
// Worse: futures inside `futs` are not polled while the body awaits
while let Some(_) = futs.next().await {
    do_thing(lock.clone()).await;
}
```

Fixes: spawn the futures (a `JoinSet` keeps every task polled by the runtime); or push the extra work into the set instead of awaiting it in the body. `join_all` keeps polling all futures, so it is not affected. A fair mutex does not cause futurelock and an unfair one does not fix it; `send_timeout` does not fix it either, because the timed-out future still has to be polled.

**Owned vs `&mut` in `select!`:** an owned future that loses is dropped (cancel-safety risk). A `&mut` future that loses stays alive but unpolled (futurelock risk if anything else in the task waits on what it holds). Either way, look hard at every `select!`.

### 3.6 `try_join` cancels side effects halfway

```rust
// Worse: one failure cancels the other flush / stop / delete midway
tokio::try_join!(stop_service_a(), stop_service_b())?;

// Better: let every operation finish, then report errors
let (a, b) = tokio::join!(stop_service_a(), stop_service_b());
a?; b?;
```

`try_join` is right for independent reads where the rest is useless after one failure; wrong for operations with side effects.

### 3.7 `abort()` as normal control flow

`abort()` can stop a task at *any* `.await`. Very little code is written to survive that. Treat aborts like panics: for shutdown, use a cooperative signal (`CancellationToken`) the task checks at points where its state is consistent.

### 3.8 Cancellation from far away

A framework that drops the request future when the client disconnects lets a remote peer cancel your code at any await. Run handlers to completion in their own task (or use a framework mode that does), and finish in-flight requests on shutdown.

### 3.9 Cleanup that never ran

A connection returned to a pool mid-transaction stays mid-transaction. Do not trust callers' futures to finish: clean up or validate resources at checkout/checkin (roll back an open transaction, reset state).

### 3.10 A future that was never awaited

`let _ = save(record);` never runs. Futures are lazy; `let _ =` silences the warning. Clippy's `let_underscore_future` catches it.

---

## 4. Tokio cancel-safety table

| Cancel safe | Mostly cancel safe (lose queue place) | Not cancel safe (lose data or progress) |
|-------------|---------------------------------------|-----------------------------------------|
| `mpsc::Receiver::recv`, `broadcast::Receiver::recv`, `watch::Receiver::changed` | `Mutex::lock`, `RwLock::read`/`write` | `mpsc::Sender::send` (owns the value) |
| `TcpListener::accept`, `UnixListener::accept`, `signal::ctrl_c` / `Signal::recv` | `Semaphore::acquire` | `AsyncReadExt::read_exact`, `read_to_end`, `read_to_string` |
| `AsyncReadExt::read`, `read_buf`; `AsyncWriteExt::write`, `write_buf` | `Notify::notified` | `AsyncWriteExt::write_all` |
| `StreamExt::next` (if the stream itself is cancel safe) | `mpsc::Sender::reserve` | `futures::SinkExt::send` |
| `time::sleep`, `JoinSet::join_next`, `CancellationToken::cancelled` | | Your own async fn that does work between two awaits |

To judge your own function: look at each `.await`. If the function is still correct when restarted while paused at that await, it is cancel safe.

---

## 5. Designing cancel-safe APIs

- **Split prepare from commit:** expose `reserve()` / `acquire()` / `poll_ready()` and a synchronous `send`.
- **Report progress through `&mut`:** take `&mut impl Buf`, or keep progress on `self`.
- **Name honestly:** `next()` / `recv()` should be cancel safe; reserve `reserve()` / `acquire()` for mostly-cancel-safe steps; give unrepeatable actions action names (`post_data`). A method taking `self` by value cannot be put in a `select!` loop by accident.
- **Document it:** add a "Cancel safety" section to public async methods.
- **Isolate with a task:** a library can run cancel-unsafe work in a spawned task to offer a cancel-safe interface.
- **Hand-written futures** (`impl Future`) must release resources correctly in `Drop`; async/await code built on them relies on it.

---

## Sources

- https://rfd.shared.oxide.computer/rfd/0400 — Dealing with cancel safety in async Rust (Oxide)
- https://rfd.shared.oxide.computer/rfd/0397 — Challenges with async/await in the control plane (Oxide)
- https://rfd.shared.oxide.computer/rfd/0609 — Futurelock (Oxide)
- https://sunshowers.io/posts/cancelling-async-rust/ — Cancelling async Rust (Rain)
- https://matklad.github.io/2026/08/31/cancelation-terminology.html — Cancelation Terminology (matklad)
- https://blog.yoshuawuyts.com/async-cancellation-1/ — Async Cancellation I (Yoshua Wuyts)
- https://blog.yoshuawuyts.com/async-cancellation-2/ — Async Cancellation II (Yoshua Wuyts)
- https://docs.rs/tokio/latest/tokio/macro.select.html — `tokio::select!` (cancellation safety section)
