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

`loop { select! { … } }` — or `merge` followed by `while let Some(event) = events.next().await` — handles **one event per wakeup**. matklad calls this a *scalar select*. The events come from outside the process (a user typing, clients sending requests, background jobs finishing), and handling one takes time. So when a step ends, several events are often already waiting in memory. A scalar loop takes an arbitrary one and pays the full per-event cost for each.

Taking everything that is already waiting in one go allows four things a scalar loop cannot do. Example: a language server with file edits, completion requests, and compiler output.

| What batching allows | What it means | Language server example |
|---|---|---|
| **Prioritization** | Choose the order inside the batch. Rule of thumb: writes before reads before accepting new clients | Apply the edit before answering the completion, so the answer sees the new text |
| **Selective backpressure** | Stop taking one kind of input for a while; the others continue | Stop accepting new clients while overloaded; their channel fills and the acceptor waits |
| **Elimination** | Drop events that a later event makes pointless | A whole-file replace drops earlier edits to the same file; an edit drops compiler output for that file, which described the old text |
| **Coalescing** | Combine events that cannot be dropped | Apply two edits, then start one diagnostics job instead of two |

Under low load each batch holds one event, so nothing is lost. As load grows the batches grow, and every per-batch cost (a job, a flush, a write, a lock) is paid once per batch instead of once per event: the work grows slower than the load.

Batching waits for the **first** event, then takes only what is already there; it never waits for more. Waiting on a timer to collect more (`tokio_stream::StreamExt::chunks_timeout`) is a different trade: it adds latency to every event to get bigger batches.

**One channel.** `Receiver::recv_many` is exactly this step: it waits for at least one message, then takes up to `limit` that are already queued, and it is cancel safe. For a merged stream, `futures::StreamExt::ready_chunks(limit)` does the same. Keep the batch decisions in a pure function, so tests pass in a batch and check the plan with no channel and no runtime (`skeptic-testability`):

```rust
/// Turns everything that arrived during the last step into the work for this step.
///
/// Writes before reads: edits, then compiler output, then requests, so answers see the latest
/// text. Each source keeps its own order. Two kinds of event become pointless and are dropped:
/// edits to a file that a later whole-file replace overwrites, and compiler output for a file
/// edited in this batch.
pub fn plan_batch(batch: impl IntoIterator<Item = Event>) -> Plan {
    let mut plan = Plan::default();
    for event in batch {
        match event {
            Event::Edit(edit) => {
                if let Edit::Replace { file, .. } = &edit {
                    plan.edits.retain(|earlier| earlier.file() != *file);
                }
                plan.edits.push(edit);
            }
            Event::CompilerOutput(diagnostics) => plan.diagnostics.push(diagnostics),
            Event::Request(request) => plan.requests.push(request),
        }
    }
    // Compiler output in this batch comes from a job started before these edits,
    // so for an edited file it describes old text.
    plan.diagnostics.retain(|diagnostics| {
        plan.edits.iter().all(|edit| edit.file() != diagnostics.file)
    });
    plan
}

/// Most events taken in one step. A full batch delays its last event by the time the step
/// takes: at about 1 ms per event, 256 events is about a quarter of a second.
const MAX_EVENTS_PER_BATCH: usize = 256;

async fn execute(server: &mut Server, plan: Plan) {
    if !plan.edits.is_empty() {
        server.apply_edits(plan.edits);
        server.start_diagnostics(); // one job for the whole batch, not one per edit
    }
    server.publish(plan.diagnostics);
    for request in plan.requests {
        server.answer(request).await;
    }
}

pub async fn run(server: &mut Server, mut events: mpsc::Receiver<Event>) {
    let mut batch = Vec::with_capacity(MAX_EVENTS_PER_BATCH);
    while events.recv_many(&mut batch, MAX_EVENTS_PER_BATCH).await > 0 {
        execute(server, plan_batch(batch.drain(..))).await;
    }
}
```

Test the rules through `plan_batch` (a replace drops earlier edits to that file; an edit drops compiler output for that file) and the order through `run` with a fake server: a request queued *before* an edit is answered against the edited text, and three queued edits start one diagnostics job.

**A second channel** for the input that must be stopped on its own (selective backpressure). Everything else stays on one channel; only new clients get their own, so the loop can stop reading it while overloaded:

```rust
/// Most new clients taken in one step. Accepting is the lowest priority, so a burst of
/// connections must not make a step much longer.
const MAX_CLIENTS_PER_BATCH: usize = 16;

pub async fn run_with_clients(
    server: &mut Server,
    mut events: mpsc::Receiver<Event>,
    mut clients: mpsc::Receiver<Client>,
) {
    let mut batch = Vec::with_capacity(MAX_EVENTS_PER_BATCH);
    let mut new_clients = Vec::with_capacity(MAX_CLIENTS_PER_BATCH);
    let mut clients_open = true;
    loop {
        // While overloaded nobody reads `clients`: its bounded channel fills and the
        // acceptor waits, while edits and requests keep flowing.
        let accepting = clients_open && !server.is_overloaded();
        tokio::select! {
            received = events.recv_many(&mut batch, MAX_EVENTS_PER_BATCH) => {
                if received == 0 {
                    break; // every event sender is gone: shut down
                }
            }
            received = clients.recv_many(&mut new_clients, MAX_CLIENTS_PER_BATCH), if accepting => {
                clients_open = received > 0; // a closed channel stays ready; stop polling it
            }
        }
        // Take what the other channel already holds, without waiting.
        while batch.len() < MAX_EVENTS_PER_BATCH {
            let Ok(event) = events.try_recv() else { break };
            batch.push(event);
        }
        while accepting && new_clients.len() < MAX_CLIENTS_PER_BATCH {
            let Ok(client) = clients.try_recv() else { break };
            new_clients.push(client);
        }
        execute(server, plan_batch(batch.drain(..))).await;
        for client in new_clients.drain(..) {
            server.accept(client);
        }
    }
}
```

Things to check in a batching loop:

- **Order:** reorder only across sources whose relative order does not matter; keep each source's own order (two edits to one file must apply in arrival order).
- **Cap:** the batch limit bounds the work of one step and how long the last event in a batch waits. Size it like any other bound (`lifetimes-and-load.md` § 4.2). The `try_recv` drains need the cap too, or a busy producer keeps one step going.
- **Closed channels:** a closed channel's `recv_many` returns 0 at once, every time. Decide per channel what closing means — shut down, or stop polling it — instead of one `break` for all, which stops the server when only the acceptor goes away.
- **Elimination must be safe:** drop an event only when a later one fully replaces its effect, and still answer anything that expects a reply (a dropped request still needs an error or a cancel response).
- **Batch size is the load signal:** batches of one mean the loop keeps up; batches at the cap mean it does not. Record the size per step (a histogram) rather than a count of events.

**Flag** a scalar loop when handling an event has a fixed cost per event (starting a job, a flush, a write, a lock, a syscall), when events can replace or combine with each other, or when some inputs must win over others. Not a finding: events that are independent, cheap, and equally urgent.

### 2.8 A task that sleeps between steps cannot react

```rust
// Worse: during each sleep the task cannot see a button press or a stop request,
// and its state ("green, 12 s in") exists only as the line it is paused on
loop {
    show(Color::Green);
    sleep(GREEN).await;
    show(Color::Yellow);
    sleep(YELLOW).await;
    show(Color::Red);
    sleep(RED).await;
}

// Better: the state holds the time of the next change; wait for whichever comes first
let mut light = Light::start(Instant::now());
show(light.color);
loop {
    let event = tokio::select! {
        _ = stop.cancelled() => break,
        Some(()) = buttons.recv() => Event::Button,
        _ = tokio::time::sleep_until(light.change_at) => Event::Tick,
    };
    let (next, command) = light.step(event, Instant::now());
    light = next;
    if let Some(command) = command {
        execute(command);
    }
}
```

`sleep` turns time into *where the program is*; the better version keeps it as data the task holds. Creating `sleep_until(light.change_at)` again on every iteration is correct here: the deadline is absolute and lives in the state, unlike the fresh relative `sleep` in § 2.1, which restarts on every message. The state and its pure `step` are in `skeptic-architecture/references/type-driven.md` § Example E2.

**Flag** `sleep(…).await` used as the wait between steps of a long-running task that must react to events or to a stop signal.

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
- https://docs.rs/tokio/latest/tokio/sync/mpsc/struct.Receiver.html#method.recv_many — `Receiver::recv_many`
- https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.ready_chunks — `StreamExt::ready_chunks`
- https://rfd.shared.oxide.computer/rfd/0609 — Futurelock (Oxide)
- https://sunshowers.io/posts/nextest-and-tokio/ — How (and why) nextest uses tokio (Rain)
