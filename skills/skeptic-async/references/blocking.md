# Blocking the runtime

Reference for reviewing whether async code stops the runtime from running other tasks. Tokio-first.

---

## 1. The model

Tokio runs many tasks on a few worker threads (one per core by default). It can only switch tasks when the running task reaches an `.await` that returns `Pending`. **Blocking** means running a long time without reaching such an `.await`: every other task scheduled on that thread waits.

- **Budget:** aim for no more than about **10–100 µs between `.await`s**; it depends on the application. Tokio is built for tasks that run microseconds to tens of milliseconds between yields.
- **No objective line:** `x + y` never blocks and `thread::sleep(10s)` always does; a sha256 of a large file or a `println!` to a slow pipe is in between. The working question is *how long does this run without yielding?*
- **Sync code called from async code is async context too** — including `Drop` impls of values dropped inside async code.
- **Local testing hides it:** with one thread per core, a few blocking tasks still look concurrent until load runs out of threads. A single-threaded `join!` exposes it at once.
- **Tokio will not save you:** it does not detect blocking tasks or add threads. Its automatic budget (128 Tokio resource operations per task before forcing a yield) only fixes always-ready IO loops; it cannot preempt CPU work or a blocking call.

---

## 2. Failure modes

### 2.1 `std::thread::sleep` and other blocking waits

```rust
// Worse: three tasks "in parallel" take 3 s — the thread is never released
async fn sleep_then_print(timer: i32) {
    std::thread::sleep(Duration::from_secs(1));
    println!("timer {timer} done");
}

// Better: yields at .await; all three finish in ~1 s
async fn sleep_then_print(timer: i32) {
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("timer {timer} done");
}
```

Same family: `std::fs`, blocking database drivers (diesel), `std::net`, `std::sync::mpsc::recv`, `crossbeam::channel::recv`, `reqwest::blocking`, waiting on a `std::thread::JoinHandle`.

### 2.2 CPU-heavy work on a worker thread

```rust
// Worse: hashing a large file on the runtime thread
async fn checksum(bytes: Vec<u8>) -> [u8; 32] { sha256(&bytes) }

// Better (expensive or parallel CPU work): rayon + oneshot, awaited
async fn checksum(bytes: Vec<u8>) -> [u8; 32] {
    let (tx, rx) = tokio::sync::oneshot::channel();
    rayon::spawn(move || { let _ = tx.send(sha256(&bytes)); });
    rx.await.expect("rayon task panicked")
}
```

Cheap work (summing a small `Vec`) can run inline. Never call `par_iter()` directly from async code — parallel iterators block the caller until they finish; wrap them in `rayon::spawn`.

### 2.3 `spawn_blocking` misused

`spawn_blocking` runs a closure on a separate, large pool (hundreds of threads by default) meant for **blocking work that finishes on its own**.

| Misuse | Why it hurts | Instead |
|--------|--------------|---------|
| An endless loop or a long-lived worker | Takes a pool thread forever; enough of them starve other blocking calls | `std::thread::spawn` |
| Many CPU-bound jobs at once | Runs far more threads than cores | `rayon`, or bound with a `Semaphore` |
| Expecting `abort()` to stop it | Started blocking tasks cannot be aborted | Pass a stop flag/channel the closure checks |
| Expecting shutdown to be quick | Runtime shutdown waits for every started blocking task | `Runtime::shutdown_timeout` (the tasks still keep running) |

```rust
// Bounded CPU work on spawn_blocking
let permit = cpu_limit.clone().acquire_owned().await?;
let digest = tokio::task::spawn_blocking(move || {
    let _permit = permit;
    sha256(&bytes)
}).await?;
```

### 2.4 `block_on` inside async code

`Runtime::block_on` and `Handle::block_on` **panic** when called from an async context (inside `#[tokio::main]`, another `block_on`, or a task). Do not wrap async code in `block_on` to call it from a sync helper that runs on the runtime.

```rust
// Worse: panics — already inside the runtime
fn load_config_sync() -> Config { Handle::current().block_on(load_config()) }
async fn handler() { let cfg = load_config_sync(); }

// Better: stay async
async fn handler() { let cfg = load_config().await; }

// When a sync callback on a multi-thread runtime truly must run async code:
tokio::task::block_in_place(|| Handle::current().block_on(load_config()));
// or hand the work to a plain thread with a Handle:
let handle = Handle::current();
std::thread::spawn(move || handle.block_on(load_config()));
```

`block_in_place` works only on the multi-thread runtime: it turns the current worker into a blocking thread and moves its other tasks elsewhere.

### 2.5 Always-ready loops

A socket echo loop under load may find `read` and `write` always ready and never yield. Tokio's coop budget handles Tokio resources automatically; loops over non-Tokio futures or sub-schedulers (e.g. `FuturesUnordered`) may not be covered — add `tokio::task::yield_now().await` in long loops.

### 2.6 Hidden blocking in `Drop`

A `Drop` that flushes a file, joins a thread, or takes a contended lock blocks the runtime when the value is dropped in async code. Give the type an explicit `async fn close(self)` and call it.

---

## 3. Which tool for which work

| Work | Use | Avoid |
|------|-----|-------|
| Any waiting (timers, sockets, channels) | The async version: `tokio::time`, `tokio::net`, `tokio::sync` | Blocking versions |
| Cheap computation (≲ 100 µs) | Run inline | Offloading — costs more than it saves |
| Bounded blocking IO (files, blocking DB driver) | `spawn_blocking` | rayon |
| A few CPU-heavy computations | `spawn_blocking` (OK) or rayon | |
| Many CPU-heavy computations | rayon + `oneshot`, or `spawn_blocking` behind a `Semaphore` | Unbounded `spawn_blocking` |
| Work that runs forever or for very long | `std::thread::spawn` | `spawn_blocking`, rayon |
| Blocking from async without moving data (multi-thread runtime only) | `block_in_place` | `block_on` |
| Long async loop that never yields | `yield_now().await` | |

---

## 4. Bridging sync and async

Calling async from sync (the program is not async at the top):

| Pattern | When | Note |
|---------|------|------|
| Keep a runtime, call `block_on` per operation | A sync wrapper around an async client, one call at a time | `current_thread` is enough; spawned tasks freeze between `block_on` calls |
| Keep a runtime, `spawn` onto it | Background work while a sync thread (e.g. a GUI) does other things | Needs `multi_thread`, or nothing runs until the next `block_on` |
| Run the runtime on its own thread, send it messages | Most flexible; the runtime is an actor | Use `blocking_send` from the sync side |

Calling sync from async: `spawn_blocking` returning a value; for streams, an `mpsc` channel with `blocking_send` on the sync side and `.recv().await` on the async side; for byte streams, `tokio_util::io::SyncIoBridge`.

Hand-built runtimes need `enable_all()` (or the specific drivers), or IO and timers fail.

---

## 5. Spawning requirements

- **`'static`:** a spawned task may outlive the caller, so it cannot borrow the caller's locals. Use `async move` with owned data, `Arc` for shared data.
- **`Send`:** a task may move between threads at any `.await`, so everything **held across an `.await`** must be `Send`. `Rc`, `RefCell` references, `std::sync::MutexGuard` held across an await make the future `!Send`. The compiler reports it at the `tokio::spawn` call, often far from the cause — fix it where the value is held, by ending its scope before the await.
- **`!Send` futures** run on a `LocalSet` / `spawn_local`.
- **Thread-locals** are not task-locals: a value read from a thread-local before an `.await` may be used on another thread after it. Use `tokio::task_local!`.
- **Recursion:** a recursive `async fn` must box the recursive call (`Box::pin`).

---

## Sources

- https://ryhl.io/blog/async-what-is-blocking/ — Async: What is blocking? (Alice Ryhl)
- https://blog.yoshuawuyts.com/what-is-blocking/ — Will it block? (Yoshua Wuyts)
- https://tokio.rs/tokio/topics/bridging — Bridging with sync code (Tokio)
- https://tokio.rs/blog/2020-04-preemption — Reducing tail latencies with automatic cooperative task yielding (Tokio)
- https://tokio.rs/tokio/tutorial/spawning — Spawning (Tokio tutorial)
- https://tokio.rs/tokio/tutorial/async — Async in depth (Tokio tutorial)
- https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html — `spawn_blocking`
- https://docs.rs/tokio/latest/tokio/task/index.html — `tokio::task`
- https://docs.rs/tokio/latest/tokio/runtime/struct.Handle.html — `Handle::block_on`
- https://matklad.github.io/2023/12/10/nsfw.html — Non-Send Futures When? (matklad)
- https://without.boats/blog/let-futures-be-futures/ — Let futures be futures (without.boats)
