# Skeptic Async

## Intent

Review async Rust (Tokio-first) for the failures the compiler does not catch: futures cancelled mid-operation, futurelock, blocking the runtime, unsafe sharing of state across `.await`, tasks with no owner, unbounded or unsized queues, actors that mix state with IO, and slow async I/O on hot paths. Runs standalone, or as an extra section of a skeptic review when the diff contains async code.

## Triggers

- **SHOULD** apply when the user asks for an async Rust review, or when skeptic runs and the diff contains `async`, `.await`, `tokio::`, futures, or a hand-written `Future`.
- **SHOULD NOT** apply to diffs with no async code.

## Behaviors

### Behavior: Cancellation points and cancel safety

The agent SHALL find every cancellation point that reaches changed code (`select!`, `timeout`, `try_join*` and `try_*` stream combinators, `run_until_cancelled`, `abort`, runtime shutdown, client disconnect, cancelled parents), SHALL read each changed async function as the cancelled future, the parent, the runtime, and other tasks, and SHALL flag futures that lose data, leave state broken, or cancel side effects halfway when dropped there, with the smallest fix.

#### Scenario: Send inside select

- **GIVEN** a `select!` loop with a branch `tx.send(value)` and a branch `interval.tick()`
- **WHEN** reviewing async code
- **THEN** the agent flags that `value` is lost when the tick wins and suggests `tx.reserve()` followed by a synchronous `permit.send(value)`

#### Scenario: Tokio mutex across an await with a broken invariant

- **GIVEN** code that takes a `tokio::sync::Mutex`, calls `take()` on the guarded `Option`, awaits, then restores it
- **WHEN** reviewing async code
- **THEN** the agent flags that cancellation at the await leaves `None` behind with no poisoning, and offers an actor or a std mutex not held across the await

#### Scenario: Batch read recreated in a select loop

- **GIVEN** an `async fn` that reads a batch from a file and sends each parsed item with `tx.send(item).await`, called as one branch of a `loop { select! { … } }` next to `socket.read_packet()`
- **WHEN** reviewing async code
- **THEN** the agent flags that each packet drops the future mid-batch and loses the remaining items silently, and suggests merging both sources as streams, or keeping pending items on `self` with reserve-then-send

#### Scenario: try_for_each_concurrent over external deletes

- **GIVEN** `try_for_each_concurrent(8, uninstall)` over a list of zones, where `uninstall` changes state outside the process
- **WHEN** reviewing async code
- **THEN** the agent flags that the first error cancels the other uninstalls midway, and suggests running all of them to the end and then reporting the first error (`join!` / `buffer_unordered` + collect, or `for_each_concurrent_then_try`)

#### Scenario: Cleanup in the cancelled arm of run_until_cancelled

- **GIVEN** a batching loop wrapped in `token.run_until_cancelled(work)`, with the `None` arm meant to flush the partial batch
- **WHEN** reviewing async code
- **THEN** the agent flags that the batch was dropped with `work` before `None` is returned, and suggests a `select!` whose stop arm breaks out to a flush that still owns the batch

### Behavior: Cancel safety marked and tested

The agent SHALL flag new or changed public async methods that callers will race or time out without a `# Cancel safety` doc section, cancel-unsafe methods named like cancel-safe ones, and cancel-unsafe code placed under a cancellation point (or documented cancel safe) without a test that drops it at an await.

#### Scenario: Documented claim with no test

- **GIVEN** a new `pub async fn flush(&mut self, tx: &Sender<T>)` documented "cancel safe" that pops an item and then awaits `tx.send(item)`, with no test that cancels it
- **WHEN** reviewing async code
- **THEN** the agent flags that the popped item is lost on cancel despite the doc, suggests reserve-then-pop, and suggests a test that polls once with a full channel (`now_or_never`), then finishes and asserts every item arrives once in order

### Behavior: Batching instead of scalar select

The agent SHALL flag an event loop that handles one event per wakeup when handling has a fixed cost per event, events can replace or combine, or some inputs must win over others, and SHALL check a batching loop for a cap on every drain, per-channel handling of closed channels, and per-source order.

#### Scenario: Every edit starts its own diagnostics job

- **GIVEN** a language server loop `while let Some(event) = rx.recv().await { process(event).await }` where each edit event starts a diagnostics job
- **WHEN** reviewing async code
- **THEN** the agent flags that edits queued during one step each start a job, and suggests `recv_many` with a stated cap, a pure `plan_batch` that orders writes before reads and drops superseded events, and one diagnostics job per batch

#### Scenario: One break for every closed channel

- **GIVEN** a batching loop over an events channel and a clients channel that breaks out of the loop when any `recv_many` returns 0
- **WHEN** reviewing async code
- **THEN** the agent flags that the server stops when only the acceptor goes away, and suggests shutting down when the events channel closes and only disabling the clients branch when the clients channel closes

#### Scenario: Task sleeps between steps

- **GIVEN** a long-running task that shows a color, awaits `sleep(30 s)`, shows the next color, and awaits again, with a button channel it must react to
- **WHEN** reviewing async code
- **THEN** the agent flags that the task cannot see the button or a stop signal while sleeping, and suggests keeping the change time in the state and selecting on `sleep_until(change_at)`, the button, and the stop token

### Behavior: Futurelock

The agent SHALL flag a task that stops polling a future that may hold a resource another future in the same task needs.

#### Scenario: Borrowed future in select plus await on the same lock

- **GIVEN** `select!` over `&mut op1` and a sleep, where the sleep branch awaits the same lock `op1` uses
- **WHEN** reviewing async code
- **THEN** the agent flags futurelock and suggests spawning `op1` and selecting on its `JoinHandle`

### Behavior: Blocking the runtime

The agent SHALL flag blocking calls and long CPU work on runtime threads, and `block_on` inside the runtime, and SHALL name the right tool for the work.

#### Scenario: Endless loop on spawn_blocking

- **GIVEN** `spawn_blocking` wrapping a loop that listens on a channel forever
- **WHEN** reviewing async code
- **THEN** the agent flags that it permanently takes a blocking-pool thread and suggests `std::thread::spawn`

### Behavior: Mutexes in async code

The agent SHALL flag a lock held across an `.await`, including the cases the compiler does not catch (`spawn_local`, `block_on` and the `#[tokio::main]` body, and `Send` guards from `dashmap` or `parking_lot` with `send_guard`), SHALL prefer a wrapper struct that locks only in non-async methods, and SHALL flag `tokio::sync::Mutex` where no await happens under the lock.

#### Scenario: dashmap entry held across an await

- **GIVEN** an `async fn` that takes a `dashmap` entry with `get_mut`, then awaits a remote call before writing to it
- **WHEN** reviewing async code
- **THEN** the agent flags a possible deadlock the compiler will not report, because `dashmap` guards are `Send`, and suggests reading and writing through sync methods with the await in between

#### Scenario: Arc<Mutex> passed through async handlers

- **GIVEN** handlers that each receive `Arc<tokio::sync::Mutex<HashMap<…>>>`, lock it, and only insert or read without awaiting
- **WHEN** reviewing async code
- **THEN** the agent suggests a wrapper struct around a `std::sync::Mutex` with non-async `insert` and `get` methods, since no await happens under the lock

### Behavior: Sans-IO actors

The agent SHALL treat an actor as a job struct plus private state: the state SHALL NOT own channel handles or sockets, `step` / `handle_message` SHALL NOT `.await`, spawn SHALL belong to the caller, and tests SHALL call use-case methods without spawn-then-sleep. On new or changed actors the agent SHALL flag the opposite, and SHALL flag sender cycles, bounded `send().await` cycles, heartbeat-only actors, and in-process actor frameworks. The agent SHALL NOT report a public `step` or a free `pub fn` that re-passes `Sender` as an async finding (architecture, component API).

#### Scenario: Actor owns the mailbox

- **GIVEN** a public struct with both `next_id` and `mpsc::Receiver<Request>` as fields, and a `run(&mut self)` loop
- **WHEN** reviewing a new or changed actor
- **THEN** the agent flags that the state holds a runtime resource, and suggests a job struct (`UniqueIds`), an inbox token the edge passes to `UniqueIds::serve`, and private state with a sync `step`

#### Scenario: Spawn in the job-struct constructor

- **GIVEN** `UniqueIds::new()` that creates a channel and `tokio::spawn`s `serve`, dropping the `JoinHandle`
- **WHEN** reviewing a new or changed actor
- **THEN** the agent flags a detached task (high) and suggests `new` return `(UniqueIds, UniqueIdsInbox)` so the caller owns `tokio::spawn(UniqueIds::serve(inbox))`

#### Scenario: Method folds the oneshot

- **GIVEN** a job-struct method that sends a request with a oneshot and `.await`s the reply before returning
- **WHEN** reviewing async code
- **THEN** the agent flags that callers cannot join several replies, and suggests returning the `oneshot::Receiver` from the use-case method (`issue_id`)

#### Scenario: Actor test waits with sleep

- **GIVEN** a test that `tokio::spawn`s the actor, sends messages, then `sleep`s before asserting
- **WHEN** reviewing async code
- **THEN** the agent flags timing-dependent coverage and suggests calling the use-case methods, dropping the job struct, awaiting `UniqueIds::serve(inbox)`, and asserting on replies

#### Scenario: Heartbeat-only actor

- **GIVEN** a task whose only job is to tick a timer and send a heartbeat while other actors do the work
- **WHEN** reviewing async code
- **THEN** the agent flags a false liveness signal and suggests the tick live in the actor whose health it represents

### Behavior: Task ownership and shutdown

The agent SHALL flag detached spawns and ignored `JoinHandle` results, and SHALL check that long-running tasks get a stop signal and that shutdown waits for them.

#### Scenario: Fire-and-forget email

- **GIVEN** a request handler that calls `tokio::spawn(async move { send_email(body).await.unwrap() })` and drops the handle
- **WHEN** reviewing async code
- **THEN** the agent flags that panics are lost and shutdown can neither stop nor wait for the work, and suggests an owned worker with a bounded queue, or a `JoinSet`/`TaskTracker` owned by the caller

#### Scenario: Helper that spawns hides running work

- **GIVEN** a library method `pub fn start_refresh(&self)` that calls `tokio::spawn` and returns `()`
- **WHEN** reviewing async code
- **THEN** the agent flags that no call site can see the work is still running, and suggests taking a `&TaskTracker` (or `&mut JoinSet`) and a stop signal from the caller

#### Scenario: Guard dropped while a spawned task uses it

- **GIVEN** a function that creates a `TempDir`, clones its path into a `tokio::spawn`ed task, and returns
- **WHEN** reviewing async code
- **THEN** the agent flags that the directory is deleted while the task still writes to it, and suggests moving the guard into the task

#### Scenario: Spawn in Drop

- **GIVEN** a `Drop` impl that takes a database handle and `tokio::spawn`s its async close
- **WHEN** reviewing async code
- **THEN** the agent flags cleanup with no owner that can race a reopen, and suggests an explicit `async fn close(self)` awaited on every exit path

### Behavior: Stop signals

The agent SHALL check how each long-running task is told to stop: a cooperative signal checked at consistent points, a matching wait, and a shutdown deadline with a stated decision for what still runs after it.

#### Scenario: Dropped handle as a stop signal

- **GIVEN** code that keeps a task's `JoinHandle` in a struct and drops the struct to stop the task
- **WHEN** reviewing async code
- **THEN** the agent flags that dropping a `JoinHandle` detaches the task, and suggests a `CancellationToken` plus waiting on the handle, or `AbortOnDropHandle` when the task is scoped to the struct and abrupt stop is acceptable

#### Scenario: Broadcast shutdown subscribed late

- **GIVEN** a shutdown `broadcast::Sender<()>` where workers call `subscribe()` when they start, possibly after shutdown was sent
- **WHEN** reviewing async code
- **THEN** the agent flags that a late subscriber never sees the signal and suggests a `CancellationToken`, whose `cancelled()` completes at once when already cancelled

### Behavior: Load and backpressure

The agent SHALL flag unbounded queues and fan-out, backpressure that does not reach the producer, and remote calls without timeouts.

#### Scenario: Unbounded channel to a slow consumer

- **GIVEN** a producer that sends every incoming event into `unbounded_channel()` consumed by a slower task
- **WHEN** reviewing async code
- **THEN** the agent flags unbounded memory growth and suggests a bounded channel with a stated capacity and a policy when full

### Behavior: Sizing queues and caps with Little's Law

The agent SHALL check every new or changed channel capacity, concurrency cap, or pool size with Little's Law (L = λ × W), SHALL ask for peak arrival rate and per-item time when the code does not state them, and SHALL flag a consumer that cannot keep up on average, a capacity or cap too small for the stated load, and a capacity with no stated memory cost.

#### Scenario: Magic-number channel capacity

- **GIVEN** `mpsc::channel(10_000)` with no comment or config stating the rate, the per-item time, or the item size
- **WHEN** reviewing async code
- **THEN** the agent asks for peak arrival rate and the longest consumer stall to absorb, shows the capacity as peak rate × stall plus headroom, and asks for worst-case memory (capacity × item size) and added wait (capacity / consumer rate)

#### Scenario: Concurrency cap below the load

- **GIVEN** a `Semaphore::new(16)` around a dependency that is stated to receive 2 000 req/s at 50 ms per call
- **WHEN** reviewing async code
- **THEN** the agent flags that 2 000 × 0.05 s = 100 calls must be in flight and the cap limits throughput to 320 req/s

### Behavior: Safe in context is not a finding

The agent SHALL NOT flag a cancel-unsafe future that cannot be cancelled in its context, and SHALL say why it is safe.

#### Scenario: Cancel-unsafe work in an owned task

- **GIVEN** a multi-step write running in its own spawned task whose handle is kept and never aborted
- **WHEN** reviewing async code
- **THEN** the agent reports it as safe in context and does not raise a cancellation finding

#### Scenario: Partial download lost on Ctrl-C

- **GIVEN** a CLI download loop that writes with `write_all` and is cancelled only by Ctrl-C at process exit
- **WHEN** reviewing async code
- **THEN** the agent says the lost partial write is acceptable in context and does not raise a finding

## Constraints

### Constraint: Stage boundary

The agent MUST NOT report timeouts as a hard-rule ban (hard-rules), tracing on spawned tasks (observability), or component API shape (architecture), MUST NOT ask for a runtime change without a measured reason, MUST NOT demand a `poll_transmit` / `poll_timeout` driver on a worker that only serializes one resource, MUST NOT demand a rewrite of an existing `Handle::new()` the diff did not change, and MUST NOT require the `cancel-safe-futures` dependency when the plain `join!`-then-check form works.

<!-- skillet-version: 1.7.0 -->
