# Performance of async and I/O code

Reference for reviewing hot paths and I/O-heavy async code. Tokio-first; the I/O-engine lessons come mostly from Turso (a Rust rewrite of SQLite with its own async I/O layer), the scheduling lessons from Tokio and the thread-per-core runtimes (Seastar, Glommio, Monoio).

---

## 1. The model: where async time goes

Async code is fast when each core spends its time on real work, not on coordination. The costs to watch, from most to least common in application code:

| Cost | Where it hides |
|------|----------------|
| Waiting that blocks a worker thread | Sync I/O, long CPU loops, `block_on` (see `blocking.md`) |
| Synchronization between cores | Shared `Mutex`/`RwLock` touched by every request; atomics and `Arc` refcounts on hot paths; one global counter updated by all workers (cache-line bouncing) |
| Allocation | A buffer, `Vec`, `String`, or boxed future per item or per request |
| Syscalls | One write per small item; one wake-up per completion |
| Too many tiny tasks or messages | One channel send per item between threads; one spawn per small job |

"There is no code faster than no code" — remove work before optimizing it. Measure before adding complexity (§ 6).

---

## 2. Two scheduling designs

| | **Work-stealing** (Tokio multi-thread) | **Thread-per-core, shared-nothing** (Seastar, ScyllaDB, Glommio, Monoio) |
|--|--|--|
| Idea | Each worker has a local queue; idle workers steal half of a busy worker's queue | One pinned thread per core; data split by key into shards; cores talk only by explicit messages |
| Wins when | Request costs vary, hot keys exist, shared state is unavoidable | State partitions cleanly (key → shard) and load is balanced; locks and atomics are the measured cost |
| Costs | Tasks move between threads → `Send` bounds, synchronization, cache misses after a steal | An overloaded core cannot get help; cross-shard operations are hard; needs pinned CPUs, Linux, often io_uring |
| Tasks | `Send + 'static` | Can be `!Send`; thread-locals are safe; refcounts need no atomics |

Both are "one OS thread per core"; the real difference is whether **data** is partitioned so each core has exclusive access to its state. Choose thread-per-core only for a measured reason and a plan for hot keys and uneven connections; otherwise stay on work-stealing. Tokio's own scheduler shows the cheap wins inside work-stealing: keep the next task on the same core (a "LIFO slot" runs a message's receiver while the message is hot in cache), move work in batches (steal half), and avoid atomic read-modify-writes in hot paths — with the caveat that a plain mutex is often better than clever atomics.

---

## 3. Techniques

### 3.1 Give hot state one owner

```rust
// Worse: every request on every core contends on one lock
let sessions: Arc<Mutex<HashMap<SessionId, Session>>> = …;

// Better: split by key; each shard owner (task or core) has exclusive access
let shard = &shards[hash(&id) % shards.len()];
shard.tx.send(Command::Touch { id, reply }).await?;
```

A sharded lock (`Vec<Mutex<_>>` indexed by hash) is the cheap first step; an owner task per shard removes the lock entirely.

### 3.2 Batch across every boundary

| Boundary | Instead of | Batch |
|----------|------------|-------|
| Cross-thread channel | One send per item | Send `Vec<Item>` chunks, or drain with `try_recv` after the first `recv` |
| Disk / network writes | One `write` per item | Vectored writes (`write_vectored` / `pwritev`), buffered writers |
| Many reads that are all needed | Await one at a time | Issue all, then wait for the group (`join_all` / `JoinSet` / a completion group) |
| Wake-ups | Wake per completion | Reap several completions per syscall (trade a little latency for fewer syscalls) |

### 3.3 Allocate less on the hot path

- Reuse buffers (`clear()` and refill; a pool of fixed-size buffers).
- Avoid copying to skip a prefix or write a tail: use views/slices (`bytes::Bytes`, `split_off`) instead of `memmove`.
- Keep results small: a large error type returned by value on every hot call costs memory traffic — box rare large errors so `Result` stays register-sized.
- Avoid `Arc::clone` per message; move or borrow. Watch for `Box::pin` per item in loops.

### 3.4 Keep work and data local

Keep a connection on the core that accepted it; drop and mutate objects on the thread that owns them; prefer `!Send` single-thread state where the runtime allows it (`LocalSet`, thread-per-core runtimes) over `Arc<Mutex<_>>` everywhere.

### 3.5 Isolate background from foreground

Compaction, backfill, or bulk exports on the same runtime as latency-sensitive requests can starve them (a component with more runnable tasks gets more CPU). Bound background concurrency, give it its own runtime or priority, and yield in long loops.

### 3.6 Cap concurrency by the resource that runs out

Count-based limits (`Semaphore` permits) for CPU or connections; byte-based limits (permits weighted by size) for memory — and reject single requests larger than the limit, or they wait forever.

---

## 4. Completion-based I/O: lessons from Turso

Turso runs SQLite-style queries on io_uring, Windows IOCP, synchronous `pread`/`pwrite`, an in-memory store, and the browser — with one engine. It does not use `async`/`await` inside the engine. Every I/O call returns a **completion** object; any function that may need I/O returns `IOResult::IO(completion)` instead of blocking, and the caller calls it again later. Re-entrant state machines make the second call resume where the first stopped. `IOResult::Done` / `IOResult::IO` play the role of `Poll::Ready` / `Poll::Pending`, and `IO::step()` plays the role of a reactor turn. The completion type also implements `Future`, so async bindings sit on top.

The lessons generalize to any code that hands buffers to the kernel or resumes after I/O:

1. **Functions that may do I/O return a resumable result, never block inside.** A synchronous wait inside the engine is what hung Turso in the browser, where completions are only delivered when control returns to the JavaScript event loop.
2. **Nothing non-idempotent before a yield point.** A `push`, `insert`, `+= 1`, lock acquisition, or cleanup that runs before the yield runs again on re-entry. Encode progress in the state enum and set the state *before* yielding.
3. **A repeat call must not resubmit I/O already in flight.** Track pending reads and syncs and wait on the existing operation.
4. **Buffers given to the kernel belong to the operation until it completes** — not to the caller's stack. With completion-based I/O (io_uring, IOCP) the kernel reads or writes the buffer after the call returns; Turso keeps write buffers alive on the completion object for exactly this reason.
5. **A completion's result is set exactly once**; late or duplicate signals are ignored.
6. **Publish shared state from the completion, after success** — never at submission. (A frame must not be visible to readers before its write is durable.)
7. **Retry short results with progress tracking, or turn them into errors.**
8. **Durability barriers wait for every write to finish** before the fsync; do not rely on kernel ordering flags.
9. **Wake on every progress event**, including intermediate chunks of a partial write, or a waiting task can deadlock.
10. **On error, cancel and reap in-flight siblings** before returning.
11. **Link a group's children before submitting them**, and keep the group open until it is fully built.
12. **I/O, clock, randomness, and sleep behind one injectable interface** — so tests can run the engine on a simulated disk with seeded latency, faults, and power loss.
13. **Test with a backend that forces every operation to yield**, and one that asserts the engine never drives I/O itself; simulation does not reproduce real kernel behavior, so also run real backends under injected short writes, EIO, and ENOSPC.

Tokio's `tokio::fs` runs file operations on the blocking pool, so most Tokio services do not touch io_uring directly. These lessons still apply to any code that resumes after I/O (cancel-safe state, idempotent re-entry, publishing after success) and to io_uring runtimes such as Monoio, where buffer ownership is part of the API: the buffer goes in and comes back with the result (`let (res, buf) = stream.read(buf).await`).

---

## 5. Hot-path code shape

- Keep the common case short: one up-front check that routes all special cases off the hot path.
- Keep hot and cold fields apart in frequently touched structs (a cache line is 64–128 bytes).
- Check for pending work with a cheap atomic load before taking a lock.
- Mark cold paths `#[inline(never)]`; keep per-item work out of an outer loop's I/O checks.

---

## 6. Measure

- Benchmark the hot path before and after; back out changes that do not measurably help unless they also simplify the code.
- Count syscalls (`strace -c`) and allocations, not only wall time.
- Profile CPU (flamegraphs) and the runtime (tokio-console for task states and stuck tasks).
- Test lock-free code and custom synchronization with `loom`.

---

## Sources

- https://github.com/tursodatabase/turso — Turso source (`core/io/`, `core/storage/pager.rs`, `core/state_machine.rs`)
- https://github.com/tursodatabase/turso/blob/main/docs/agent-guides/async-io-model.md — Turso async I/O model
- https://github.com/tursodatabase/turso/blob/main/PERF.md — Turso performance testing
- https://github.com/tursodatabase/turso/blob/main/docs/testing.md — Turso testing and deterministic simulation
- https://turso.tech/blog/introducing-limbo-a-complete-rewrite-of-sqlite-in-rust — Introducing Limbo (Turso)
- https://tokio.rs/blog/2019-10-scheduler — Making the Tokio scheduler 10x faster (Tokio)
- https://without.boats/blog/thread-per-core/ — Thread-per-core (without.boats)
- https://seastar.io/shared-nothing/ — Shared-nothing design (Seastar)
- https://seastar.io/message-passing/ — Message passing (Seastar)
- https://github.com/scylladb/seastar/blob/master/doc/tutorial.md — Seastar tutorial
- https://www.scylladb.com/product/technology/shard-per-core-architecture/ — Shard-per-core architecture (ScyllaDB)
- https://www.datadoghq.com/blog/engineering/introducing-glommio/ — Introducing Glommio (Datadog)
- https://github.com/bytedance/monoio — Monoio
- https://rfd.shared.oxide.computer/rfd/0445 — Crucible Upstairs Backpressure (Oxide)
