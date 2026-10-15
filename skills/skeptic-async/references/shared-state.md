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
| A database connection | A connection pool (`bb8`, `deadpool`), not a mutex around one connection |

- **Use an async lock only when you must `.await` while holding it.** Async locks are "a lot slower than blocking locks", and `tokio::sync::Mutex` uses a synchronous mutex inside, so switching to it rarely helps performance.
- A synchronous mutex in async code "is fine as long as contention remains low and the lock is not held across calls to `.await`." A contended lock blocks the thread, and with it every other task scheduled on that thread.
- `tokio::sync::Mutex` is FIFO and fair, does not poison on panic, its `lock()` loses its queue place when cancelled, and `blocking_lock` panics inside async code. It cannot be locked in `Drop`, because destructors are synchronous.

### 2.1 Hide the lock in a wrapper struct

Do not pass `Arc<Mutex<MyStruct>>` around with `lock()` calls everywhere. That clutters every signature, leaks the choice of lock (swapping `Mutex` for `RwLock` touches the whole codebase), and makes it easy to hold the lock too long — "a particularly bad issue in async code." Put the `Arc<Mutex<…>>` inside a struct, and lock only inside its **non-async** methods: "Since you can't use .await in a non-async method, this makes it impossible to accidentally .await something while the mutex is locked."

```rust
#[derive(Clone)]
pub struct SessionTable { inner: Arc<Mutex<HashMap<SessionId, Session>>> }

impl SessionTable {
    pub fn insert(&self, id: SessionId, session: Session) { self.inner.lock().unwrap().insert(id, session); }
    pub fn get(&self, id: &SessionId) -> Option<Session> { self.inner.lock().unwrap().get(id).cloned() }
}

// Async code calls the sync methods; the guard never lives across an await
async fn refresh(table: &SessionTable, id: SessionId) -> Result<(), Error> {
    let session = table.get(&id).ok_or(Error::Unknown)?;
    let renewed = auth_client.renew(session).await?;   // no lock held here
    table.insert(id, renewed);
    Ok(())
}
```

Async methods on the wrapper follow the same rule: read through a small sync helper (`fn deadline(&self) -> Instant`) and await afterwards, as in Ryhl's debouncer, whose `sleep()` never locks directly.

### 2.2 Never return a guard or a reference into the lock

A reference into the locked value exists only while the guard does, so returning one means returning the guard — and then async callers hold it across their awaits. Instead:

- **Clone the value out**, made cheap with `Arc<str>` or `Bytes` instead of `String` or `Vec<u8>`.
- **Run a closure under the lock** (`with_*` pattern): the closure is synchronous, so it cannot await.

```rust
pub fn with_session<T>(&self, id: &SessionId, read: impl FnOnce(Option<&Session>) -> T) -> T {
    let guard = self.inner.lock().unwrap();
    read(guard.get(id))
}
```

### 2.3 Alternatives to a mutex

Wrap each one in a struct the same way.

| Tool | Fits | Watch for |
|------|------|-----------|
| `RwLock` | Many more reads than writes | Writer starvation under constant readers; `parking_lot`'s `RwLock` and Tokio's locks are fair |
| `arc-swap` | A value that changes rarely (config) | Readers never lock and see the old value during a write; every write clones the whole value (or shares parts through `Arc`) |
| `evmap` | A map read far more than written | Eventually consistent: writes become visible later |
| `dashmap` | A map with independent keys | Sharded `RwLock`s; its guards are `Send`, so holding one across `.await` compiles and can deadlock (§ 3.1) |
| A thread-local | A per-thread cache | Data duplicated per thread |
| Atomics | A counter or flag | — |
| Sharded `Vec<Mutex<HashMap<…>>>` | Independent keys under contention | Pick the shard by key hash |

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

The Tokio tutorial warns that an explicit `drop(n)` before the await may not be enough, because the compiler decides `Send` from scopes. Ending the scope always works:

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

Why it deadlocks: tasks switch only at an `.await`. A task that awaits while holding a lock keeps it locked; if another task on the same thread then calls `lock()` — which is not an `.await` — the thread blocks and can never switch back to the holder.

**When the compiler does not catch it.** The `!Send` error appears only when the task can move between threads *and* the guard is not `Send`. No error, and a possible deadlock, when:

- the code runs under `tokio::task::spawn_local`, or `block_on` — which includes the body of `#[tokio::main]`;
- the guard is `Send`: **`dashmap`'s guards are**, and `parking_lot`'s are with its `send_guard` feature — which `dashmap` enables for the whole build, so every `parking_lot` lock in a crate that depends on `dashmap` loses the compiler check.

So do not "fix" the `!Send` error by moving the task to a local runtime, and lock `dashmap` and `parking_lot` only inside non-async methods (§ 2.1).

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
- **No IO resources inside a locked collection.** A `Mutex<HashMap<UserId, TcpStream>>` "never really" turns out well: give each connection its own task that owns the stream, and keep the actor *handles* in the map.

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

- https://draft.ryhl.io/blog/shared-mutable-state/ — Shared mutable state in Rust (Alice Ryhl)
- https://tokio.rs/tokio/tutorial/shared-state — Shared state (Tokio tutorial)
- https://tokio.rs/tokio/tutorial/channels — Channels (Tokio tutorial)
- https://docs.rs/tokio/latest/tokio/sync/struct.Mutex.html — `tokio::sync::Mutex` (which mutex to use)
- https://ryhl.io/blog/actors-with-tokio/ — Actors with Tokio (Alice Ryhl)
- https://matklad.github.io/2025/11/04/on-async-mutexes.html — On Async Mutexes (matklad)
- https://without.boats/blog/futures-unordered/ — FuturesUnordered and the order of futures (without.boats)
- https://rfd.shared.oxide.computer/rfd/0397 — Challenges with async/await in the control plane (Oxide)
- https://rfd.shared.oxide.computer/rfd/0609 — Futurelock (Oxide)
