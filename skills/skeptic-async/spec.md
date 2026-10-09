# Skeptic Async

## Intent

Review async Rust (Tokio-first) for the failures the compiler does not catch: futures cancelled mid-operation, futurelock, blocking the runtime, unsafe sharing of state across `.await`, tasks with no owner, unbounded or unsized queues, and slow async I/O on hot paths. Runs standalone, or as an extra section of a skeptic review when the diff contains async code.

## Triggers

- **SHOULD** apply when the user asks for an async Rust review, or when skeptic runs and the diff contains `async`, `.await`, `tokio::`, futures, or a hand-written `Future`.
- **SHOULD NOT** apply to diffs with no async code.

## Behaviors

### Behavior: Cancellation points and cancel safety

The agent SHALL find every cancellation point that reaches changed code (`select!`, `timeout`, `try_join*`, `abort`, runtime shutdown, client disconnect, cancelled parents) and SHALL flag futures that lose data or leave state broken when dropped there, with the smallest fix.

#### Scenario: Send inside select

- **GIVEN** a `select!` loop with a branch `tx.send(value)` and a branch `interval.tick()`
- **WHEN** reviewing async code
- **THEN** the agent flags that `value` is lost when the tick wins and suggests `tx.reserve()` followed by a synchronous `permit.send(value)`

#### Scenario: Tokio mutex across an await with a broken invariant

- **GIVEN** code that takes a `tokio::sync::Mutex`, calls `take()` on the guarded `Option`, awaits, then restores it
- **WHEN** reviewing async code
- **THEN** the agent flags that cancellation at the await leaves `None` behind with no poisoning, and offers an actor or a std mutex not held across the await

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

### Behavior: Task ownership and shutdown

The agent SHALL flag detached spawns and ignored `JoinHandle` results, and SHALL check that long-running tasks get a stop signal and that shutdown waits for them.

#### Scenario: Fire-and-forget email

- **GIVEN** a request handler that calls `tokio::spawn(async move { send_email(body).await.unwrap() })` and drops the handle
- **WHEN** reviewing async code
- **THEN** the agent flags that panics are lost and shutdown can neither stop nor wait for the work, and suggests an owned worker with a bounded queue, or a `JoinSet`/`TaskTracker` owned by the caller

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

## Constraints

### Constraint: Stage boundary

The agent MUST NOT report timeouts as a hard-rule ban (hard-rules), tracing on spawned tasks (observability), or component API shape (architecture), and MUST NOT ask for a runtime change without a measured reason.

<!-- skillet-version: 1.7.0 -->
