# Actors (sans-IO)

Reference for reviewing in-process Tokio actors.

The **public face** is a job struct with use-case methods (skeptic-architecture). The **interior** is data: messages in, new state and work out. A thin `serve` loop owns sockets and channels and is started at the app edge.

When to pick an actor over a mutex: `shared-state.md` §4. Spawn ownership and `CancellationToken`: `lifetimes-and-load.md`. Pure `step` in general: skeptic-testability.

---

## 1. The model

The actor's **state** is plain data. It does not own a `Receiver`, a socket, a `JoinHandle`, or a timer. Those are runtime resources: `serve` holds them, feeds the state, and does the work the state returns.

Callers never see the state type. They hold the job struct and call use cases. `step`, the request enum, and the loop stay private.

```text
sync method on a wrapper                 (no actor)
        ↓  exclusive async work on one resource
private state + sync step(request) → work
        ↓  must serialize IO
UniqueIds::serve(inbox)                  (spawn only at the edge)
        ↓  must outlive the caller
owned worker (JoinSet / TaskTracker), bounded mailbox
```

The only thing better than an actor is no actor. If a sync method on a wrapper struct would do, do not add a mailbox.

This is the same split as thinking code vs shell: decisions are data; IO stays in `serve`. A protocol can go further (`handle_input` / `poll_transmit` / `handle_timeout`). Do not demand that driver on a worker whose job is serializing one resource — demand that **`step` does not `.await`**.

---

## 2. Job struct on the outside; state is data on the inside

```rust
// Worse: one public struct mixes the job, the mailbox, and a socket —
// hard to restart, hard to test, callers see the pipeline
pub struct UniqueIds {
    inbox: mpsc::Receiver<Request>,
    socket: TcpStream,
    next_id: u32,
}

// Worse: free function that re-passes the sender on every call
pub async fn issue_id(tx: &mpsc::Sender<Request>) -> Option<oneshot::Receiver<u32>> { ... }

// Better: public face is the job. State, requests, and the loop stay private.
pub struct UniqueIds {
    tx: mpsc::Sender<Request>,
}

/// Receive half. Only `UniqueIds::serve` drives it — the edge owns when that runs.
pub struct UniqueIdsInbox {
    rx: mpsc::Receiver<Request>,
}

impl UniqueIds {
    pub fn new(mailbox_capacity: usize) -> (Self, UniqueIdsInbox) {
        let (tx, rx) = mpsc::channel(mailbox_capacity);
        (Self { tx }, UniqueIdsInbox { rx })
    }

    /// Use case. Does not wait for the id — returns the oneshot receiver.
    pub async fn issue_id(&self) -> Option<oneshot::Receiver<u32>> {
        let (reply, rx) = oneshot::channel();
        self.tx.send(Request::NextId { reply }).await.ok()?;
        Some(rx)
    }

    pub async fn closed(&self) {
        self.tx.closed().await;
    }

    /// Shell: owns the mailbox. The edge calls this (or spawns it).
    pub async fn serve(inbox: UniqueIdsInbox) {
        UniqueIdState { next_id: 0 }.run(inbox.rx).await;
    }
}

enum Request {
    NextId { reply: oneshot::Sender<u32> },
}

struct UniqueIdState {
    next_id: u32,
}

impl UniqueIdState {
    fn step(&mut self, request: Request) {
        match request {
            Request::NextId { reply } => {
                let id = self.next_id;
                self.next_id += 1;
                let _ = reply.send(id); // caller may have given up; that is fine
            }
        }
    }

    async fn run(mut self, mut rx: mpsc::Receiver<Request>) -> Self {
        while let Some(request) = rx.recv().await {
            self.step(request);
        }
        self
    }
}
```

Sample caller (composition root — wires the pieces, does not live inside `UniqueIds`):

```rust
let (ids, inbox) = UniqueIds::new(UNIQUE_ID_MAILBOX);
let join = tokio::spawn(UniqueIds::serve(inbox)); // or await `serve` without spawning
let reply = ids.issue_id().await.unwrap();
let id = reply.await.unwrap();
```

When `step` would need to wait, that IO is a **different job's** `serve`, passed in at the edge — do not hang a socket on `UniqueIds`:

```rust
pub struct ConnectionWriter { tx: mpsc::Sender<WriteRequest> }
pub struct ConnectionWriterInbox { rx: mpsc::Receiver<WriteRequest> }

enum Action {
    Write(Vec<u8>),
}

impl ConnectionWriter {
    pub async fn serve(inbox: ConnectionWriterInbox, mut sink: impl AsyncWrite) {
        let mut state = WriterState { /* … */ };
        while let Some(request) = inbox.rx.recv().await {
            if let Some(Action::Write(bytes)) = state.step(request) {
                sink.write_all(&bytes).await.ok();
            }
        }
    }
}

impl WriterState {
    fn step(&mut self, request: WriteRequest) -> Option<Action> { /* no .await */ }
}
```

Pass time in as an argument to `step`. Do not call `Instant::now()` inside `step`.

Flag on new or changed actors:

- **State struct holds a `Receiver`, socket, `JoinHandle`, or interval.** Ownership is responsibility: `serve` is responsible for those. Restart = new inbox + the same state.
- **`.await` inside `step` / `handle_message`.** The decision is no longer testable without a runtime.
- **Public `step`, public `Request`, or a free `pub fn` that takes `Sender<Request>` on every call.** The job struct is the face; thinking and the mailbox protocol stay inside. File under architecture (`### Component API`), not this section.

---

## 3. Spawn at the edge

```rust
// Worse: constructor detaches the task. Nobody owns the JoinHandle.
impl UniqueIds {
    pub fn new() -> Self {
        let (tx, rx) = mpsc::channel(8);
        tokio::spawn(UniqueIds::serve(UniqueIdsInbox { rx }));
        Self { tx }
    }
}

// Better: `new` returns the inbox; the caller owns spawn vs join of `serve`
let (ids, inbox) = UniqueIds::new(8);
let join = tokio::spawn(UniqueIds::serve(inbox));
```

Leaving spawn to the caller is structured concurrency (`lifetimes-and-load.md`): the parent can stop the actor, wait for it, and see a panic. A job-struct constructor that spawns is the same failure as fire-and-forget `tokio::spawn` in a request handler.

Flag **high** on new or changed actors. Do not demand a rewrite of every existing `Handle::new()` the diff did not touch.

---

## 4. Talking to the actor

The job struct is how callers talk. Use-case methods send requests (`issue_id`). Do not put `mpsc::Sender<Request>` on the public face: that makes `Request` public and lets callers skip the job methods.

If the caller must not wait, add `try_issue_id` on `UniqueIds`. If they need to know the worker is gone, `closed()` is enough. The full sender API stays inside.

```rust
// Worse: the method awaits the id, so the caller cannot join several replies
pub async fn next_id(&self) -> u32 {
    let (reply, rx) = oneshot::channel();
    let _ = self.tx.send(Request::NextId { reply }).await;
    rx.await.expect("actor stopped")
}
```

---

## 5. Messages

A request **owns** its data. A borrowed payload ties the sender's lifetime to the actor and defeats a `'static` task. Keep the enum private when callers go through the job struct.

Prefer **commands** that flow one way. Call-and-response (oneshot in the request) is for queries. A calling actor that `.await`s a reply while its own outbound bounded channel is full is the deadlock in §6.

`oneshot::send` and `try_send` never wait, so they do not count toward a wait-cycle.

---

## 6. Topology and shutdown

**The channel graph should be a DAG.** Two failure modes of a cycle:

1. **No natural shutdown.** `recv()` returns `None` only when every `Sender` is dropped. Actors that hold each other's senders never see `None`. An actor that holds a clone of its own sender never exits.
2. **Deadlock on bounded `send().await`.** A waits to send to B, B waits to send to A, both queues are full. Large capacities make this rarer, not impossible — test with small capacities.

`oneshot` replies and `try_send` do not wait, so they do not form this deadlock. Break a remaining cycle with a second channel whose close ends the loop (`select!` with `Some(msg) = rx.recv()`, `else => break` — `select-and-tasks.md`).

**Natural shutdown** (inside private `run`):

```rust
while let Some(request) = rx.recv().await {
    self.step(request);
}
// optional async cleanup (unsubscribe, close a socket), then drop runtime resources
```

Dropping the job struct (the last sender) shuts a DAG down. Prefer that over a `CancellationToken`. Use a token when senders must stay alive (a signal, a timeout, “stop this subtree while others still send”).

**IO connections:** one reader task and one writer task per connection. They own the stream; they are that job's `serve`. The rest of the program talks to them with messages and can pretend a write is instant. Do not put streams in a `Mutex<HashMap<…>>` (`shared-state.md` §4).

Bounded inbox, even when there is no reply. Size it (`lifetimes-and-load.md` §4.2). Fire-and-forget producers that must not wait: `try_send` (as a method on the job struct) and treat full as an error.

---

## 7. Testing

Tests use the **public job struct**, not `step` or state fields (those are interior; same-module tests may call them, production tests should not need to).

If `step` is sync, table-test it next to the state with no Tokio.

When the loop is in the test, **call the use case, drop the job struct, then await `serve`**. That is a deterministic order: no spawn, no `sleep`.

```rust
#[tokio::test]
async fn issues_sequential_ids() {
    let (ids, inbox) = UniqueIds::new(3);
    let r1 = ids.issue_id().await.unwrap();
    let r2 = ids.issue_id().await.unwrap();
    let r3 = ids.issue_id().await.unwrap();
    drop(ids); // natural shutdown
    UniqueIds::serve(inbox).await;
    assert_eq!(tokio::try_join!(r1, r2, r3).unwrap(), (0, 1, 2));
}
```

Flag:

- spawn the actor, then `sleep` / timeout until a condition — there is no real-time contract inside Tokio to make that honest
- asserts inside a test-only spawned task — join the work from the test task
- a test that can only see final state by reading private fields because the public face has no use case for the outcome (add a query, or assert on replies)

If you do spawn in production, `.await` the `JoinHandle` so a panic is not dropped (`lifetimes-and-load.md` §2.2).

---

## 8. Anti-patterns

- **Heartbeat-only actor.** Periodic work is an input to the actor whose liveness it represents (`select!` a tick in *that* loop, or `handle_timeout` on the state). A sibling that still beats while the real work is stuck is a false signal.
- **In-process actor framework** (actix, ractor, kameo, xtra, …). The framework is `mpsc` + a task. Exception: a named need for OTP-style supervision trees or distributed addressing.
- **Call-response as the default protocol.** Commands left-to-right keep the DAG and the natural shutdown. Use oneshots for queries, and do not fold the receiver.

---

## Sources

- https://ryhl.io/blog/actors-with-tokio/ — Actors with Tokio (Alice Ryhl): bounded mailbox, drop-senders shutdown, no wait-cycles, reader/writer split
- More Actors with Tokio (Flipped Bits, 2025) — state is data; consuming run loop; spawn at the edge; enqueue-then-run tests
- https://www.firezone.dev/blog/sans-io — sans-IO in Rust (Firezone): state machine vs mutex vs actors
- https://blog.yoshuawuyts.com/replacing-tasks-with-actors/ — Replacing Background Tasks With Actors (Yoshua Wuyts): join the actor with its parent
- https://sans-io.readthedocs.io/ — sans-IO (Python): the same split, independent of Tokio
