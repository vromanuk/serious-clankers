# Shared state, actors, and channels

Reference for reviewing how async code shares mutable state. Tokio-first.

---

## 1. The model

Two ways to share state between tasks:

1. **Guard it with a mutex** — for plain data with quick synchronous operations.
2. **Give it to one task and send that task messages** (an actor) — for things that need async work, such as an IO resource.

Async does not remove the need to think about exclusion. **Mutual exclusion is a property of the logic, not the runtime**: even on one thread, inserting an `.await` in the middle of code that must be atomic lets other tasks run there and see half-updated state.

---

## 2. Which mutex

| Situation | Use |
|-----------|-----|
| Plain data, short critical sections, never held across `.await` | `std::sync::Mutex` (or `parking_lot`) — preferred, cheaper |
| An IO resource that must be held across `.await` | `tokio::sync::Mutex` works, but an actor that owns the resource is usually better |
| A pipelined client (many requests in flight on one connection) | Actor — a Tokio mutex allows only one request in flight |
| A contended `std::sync::Mutex` | Shard it, move the state into an actor, or restructure — rarely switch to the Tokio mutex |

`tokio::sync::Mutex` uses a synchronous mutex inside; it does not make code faster. It is FIFO, does not poison on panic, its `lock()` loses its queue place when cancelled, and `blocking_lock` panics inside async code.

---

## 3. Failure modes

### 3.1 A std guard alive at an `.await`

```rust
// Worse: the future is !Send (spawn fails); on a local runtime it can deadlock
async fn increment_and_save(counter: &std::sync::Mutex<u64>) {
    let mut n = counter.lock().unwrap();
    *n += 1;
    save().await;
}
```

An explicit `drop(n)` before the await is **not** enough: the compiler decides `Send` from scopes. End the scope:

```rust
// Better: guard scope ends before the await
async fn increment_and_save(counter: &std::sync::Mutex<u64>) {
    { *counter.lock().unwrap() += 1; }
    save().await;
}

// Best: lock only inside non-async methods, so an await can never sit under the guard
struct Counter { n: std::sync::Mutex<u64> }
impl Counter {
    fn increment(&self) -> u64 { let mut n = self.n.lock().unwrap(); *n += 1; *n }
}
async fn increment_and_save(counter: &Counter) {
    counter.increment();
    save().await;
}
```

Do not "fix" the `!Send` error by moving the task to a local runtime: another task on that thread can then block on the same lock while the holder waits at its await — a deadlock. Some mutex crates make the guard `Send`; that code compiles and still deadlocks.

### 3.2 A Tokio mutex held while state is broken

```rust
// Worse: cancellation at the await leaves `None` behind (no poisoning, no warning)
let mut guard = state.lock().await;
let current = guard.take().unwrap();
*guard = Some(current.advance().await);
```

Prefer an actor or a std mutex not held across awaits. See `cancellation.md` § 3.4.

### 3.3 State read before an `.await` and trusted after it

```rust
// Worse: another task may have changed `balance` while we awaited
let balance = account.balance();          // read
let approved = risk_service.check().await;
account.set_balance(balance - amount);    // writes a stale value

// Better: re-read and re-check after the await, under one synchronous critical section
let approved = risk_service.check().await;
account.withdraw_if_enough(amount)?;     // sync method: read-check-write together
```

### 3.4 Unbounded queues

```rust
// Worse: a slow consumer lets memory grow without limit
let (tx, rx) = tokio::sync::mpsc::unbounded_channel();

// Better: a bound makes senders wait (backpressure)
let (tx, rx) = tokio::sync::mpsc::channel(64);
```

Every source of concurrency needs an explicit bound: spawn loops, `select!`/`join!` fan-out, channels, open sockets. Bounds are application-specific — pick one on purpose and write down why (size it with Little's Law, `lifetimes-and-load.md` §4.2).

### 3.5 Blocking channels in async code

`std::sync::mpsc` and `crossbeam::channel` block the thread on `recv`. Use `tokio::sync` channels (or `async-channel`).

---

## 4. Actors

An actor is a task that owns some state and a **handle** that other code uses to send it messages. The handle keeps the actor alive.

```rust
use tokio::sync::{mpsc, oneshot};

enum Message {
    NextId { reply: oneshot::Sender<u64> },
}

struct IdActor { inbox: mpsc::Receiver<Message>, next_id: u64 }

impl IdActor {
    fn handle(&mut self, msg: Message) {
        match msg {
            Message::NextId { reply } => {
                self.next_id += 1;
                let _ = reply.send(self.next_id); // the caller may have given up; that is fine
            }
        }
    }
}

async fn run(mut actor: IdActor) {
    while let Some(msg) = actor.inbox.recv().await { actor.handle(msg); }
    // every handle dropped → recv() returns None → actor exits
}

#[derive(Clone)]
pub struct IdHandle { tx: mpsc::Sender<Message> }

impl IdHandle {
    pub fn new() -> Self {
        let (tx, inbox) = mpsc::channel(8);
        tokio::spawn(run(IdActor { inbox, next_id: 0 }));
        Self { tx }
    }
    pub async fn next_id(&self) -> u64 {
        let (reply, response) = oneshot::channel();
        let _ = self.tx.send(Message::NextId { reply }).await; // if this fails, so will the recv
        response.await.expect("actor task stopped")
    }
}
```

Rules:
- **Separate handle and actor.** A `run(&mut self)` method that calls `tokio::spawn` cannot compile (`'static`), and merging the two lets every handle touch actor-only fields.
- **Spawn in the handle's constructor**, with a top-level `async fn run(actor)`.
- **Bounded inbox**, even when no reply is sent. For fire-and-forget producers that must not wait, `try_send` and treat a full inbox as an error (e.g. drop a writer that cannot keep up).
- **Shutdown = all handles dropped.** A cycle of actors holding each other's handles never shuts down; an actor holding a clone of its own sender never sees `None`. Break the cycle with a second channel whose closing ends the loop, or abort one actor.
- **No cycle of bounded channels.** If A awaits sending to B while B awaits sending to A and both queues are full, both wait forever. `oneshot` replies and `try_send` do not count toward a cycle (they never wait).
- **IO actors:** one reader task and one writer task per connection keep IO isolated.

---

## 5. Channels

| Channel | Shape | Use for |
|---------|-------|---------|
| `mpsc` | many senders, one receiver, bounded | Commands to an actor; work queues |
| `oneshot` | one value, once | A reply to one request |
| `broadcast` | every receiver sees every value | Fan-out events |
| `watch` | receivers see only the latest value | Config, status, shutdown flags |

- `recv()` returns `None` when every sender is dropped — that is the normal shutdown signal.
- A bounded `send().await` puts the sender in an unbounded wait queue of its own; when the latest value is all that matters, `watch` avoids queuing entirely; when the producer must not wait, use `try_send` and push the error back (e.g. HTTP 429).
- Large arbitrary capacities make deadlocks rarer, not impossible. Test with small capacities so cycles show up before production.

---

## Sources

- https://tokio.rs/tokio/tutorial/shared-state — Shared state (Tokio tutorial)
- https://tokio.rs/tokio/tutorial/channels — Channels (Tokio tutorial)
- https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html — `tokio::sync::Mutex` (which mutex to use)
- https://ryhl.io/blog/actors-with-tokio/ — Actors with Tokio (Alice Ryhl)
- https://matklad.github.io/2025/11/04/on-async-mutexes.html — On Async Mutexes (matklad)
- https://without.boats/blog/futures-unordered/ — FuturesUnordered and the order of futures (without.boats)
- https://rfd.shared.oxide.computer/rfd/0397 — Challenges with async/await in the control plane (Oxide)
- https://rfd.shared.oxide.computer/rfd/0609 — Futurelock (Oxide)
