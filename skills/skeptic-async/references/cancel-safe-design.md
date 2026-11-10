# Cancel-safe design

Reference for reviewing how async code avoids cancellation bugs: APIs that are cancel safe or clearly marked, callers that do not cancel what must finish, and tests that prove it. The failure modes themselves are in `cancellation.md`. Tokio-first.

A cancel-correctness bug needs three things: a cancel-unsafe future, a cancellation of it, and a property that the cancellation breaks (`cancellation.md` § 1). § 1 below removes the first; § 2 removes the second; § 3 checks both.

---

## 1. Writing async APIs

There is no single fix. Make as much as possible cancel safe; mark the rest so a caller feels that putting it in a `select!` is wrong.

### 1.1 Split prepare from commit

Expose an async "get ready" step that is (mostly) cancel safe and a synchronous step that does the irreversible part: `Sender::reserve()` → `Permit::send`, `Semaphore::acquire()` → work, `Sink::poll_ready` → `start_send`.

Consider not offering the one-call convenience (`send(value).await`) at all. It is the version people put in `select!` and `timeout`. Oxide's serial console proxy lost data through four futures in one `select!`; one fix was to delete a `send` wrapper and implement `Sink` instead, so callers reserve first.

### 1.2 Keep progress outside the future

If the work takes several awaits, record progress where it survives a drop: in a field on `self`, or in a `&mut` argument (`&mut impl bytes::Buf`, as `write_all_buf` does).

```rust
#[derive(Default)]
pub struct Outbox<T> {
    pending: VecDeque<T>,
}

impl<T> Outbox<T> {
    pub fn push(&mut self, item: T) {
        self.pending.push_back(item);
    }

    /// Sends every pending item, in order.
    ///
    /// # Cancel safety
    ///
    /// Cancel safe: an item leaves the outbox only once a channel slot is reserved for it,
    /// so a dropped call loses nothing and the next call resumes with the same item.
    pub async fn flush(&mut self, tx: &mpsc::Sender<T>) -> Result<(), SendError<()>> {
        while !self.pending.is_empty() {
            let permit = tx.reserve().await?;
            permit.send(self.pending.pop_front().expect("checked non-empty"));
        }
        Ok(())
    }
}
```

The version that does `while let Some(item) = self.pending.pop_front() { tx.send(item).await?; }` looks the same and loses the popped item on every cancel (§ 3 shows the test that catches it).

### 1.3 Run cancel-unsafe work in a task

A task is driven by the runtime, not by the caller, and dropping its `JoinHandle` does not stop it. A library can spawn the cancel-unsafe part and offer a cancel-safe interface on top (for example, a method that only waits for the task's result). Store the handle on `self` before the next await, so a cancelled call can pick it up. Task ownership rules still apply (`lifetimes-and-load.md` § 2.7).

### 1.4 Restore state when the future is dropped

Cancellation runs only synchronous `Drop`. If a function must break an invariant across an await, put the repair in a drop guard created before the await (a small struct with `Drop`, or `scopeguard::guard`), and disarm it on success. This is the async form of unwind safety: the same guard covers panics.

Hand-written futures (`impl Future`) must release what they hold in `Drop` — wakers registered, slots reserved, buffers lent to the kernel. Every `async fn` built on them relies on it.

### 1.5 Mark what stays cancel unsafe

**Names.**
- `next()` and `recv()` promise cancel safety; do not use them for cancel-unsafe methods.
- `reserve()` and `acquire()` promise "mostly cancel safe" (only the queue place is lost).
- Name irreversible actions after the action: `http_post_data()`, `uninstall_zone()`. Nobody expects to restart a POST every loop.

**Types.**
- A method that takes `self` by value (`async fn execute(self)`) cannot be recreated in a `select!` loop; the caller must create the future once. It can still be cancelled once (by a timeout), which usually aborts the whole operation on purpose.
- A two-phase API makes the cancel-safe part and the rest separate futures:

```rust
impl Connection {
    /// Cancel safe: returns as soon as a message is read; nothing is lost if dropped before that.
    pub async fn recv(&mut self) -> Pending<'_> { … }
}

/// A message read but not yet processed. The borrow keeps `recv` from being called again
/// until this is done or dropped.
#[must_use = "dropping a Pending discards the message"]
pub struct Pending<'a> {
    message: Message,
    _conn: PhantomData<&'a mut ()>,
}

impl Pending<'_> {
    /// Not cancel safe: run it to completion.
    pub async fn process(self) -> Message { … }
}
```

**Docs.** Give each public async method a `# Cancel safety` section, the way Tokio does: cancel safe, mostly cancel safe (what is lost), or not cancel safe (what is lost, and what to use instead). At least do it for the methods people will put in a `select!`.

---

## 2. Calling cancel-unsafe code

### 2.1 Read the signals

A method called `execute`, taking `self`, performing a POST, or documented as not cancel safe does not belong in a `select!` loop or under a `timeout` unless losing the work is fine. When a method is undocumented, judge it by its awaits (`cancellation.md` § 4).

### 2.2 Resume futures instead of recreating them

Create the future once, outside the loop, and poll it through `&mut`. A recreated `reserve()` loses its place in the queue on every loss; a resumed one keeps it:

```rust
let mut values = values.into_iter().peekable();
let mut reserve = Box::pin(tx.reserve());
while values.peek().is_some() {
    tokio::select! {
        permit = &mut reserve => {
            let Ok(permit) = permit else { break };
            permit.send(values.next().expect("peeked"));
            reserve.set(tx.reserve());   // start the next wait only after this one finished
        }
        _ = interval.tick() => println!("still waiting for channel capacity"),
    }
}
```

Use `std::pin::pin!` for a future that is never replaced; `Box::pin` plus `set` when it is. Resuming trades the cancel-safety risk for a futurelock risk if a handler awaits something the resumed future holds (`cancellation.md` § 3.5).

### 2.3 Run everything to the end, then check errors

`try_join!`, `try_join_all`, `try_for_each_concurrent`, and `try_collect` cancel the rest on the first `Err`. For work with side effects — flushes, stops, deletes, anything outside the process — run every future to the end, then report:

```rust
let (a, b) = tokio::join!(file_a.flush(), file_b.flush());
a?;
b?;
```

Oxide's `cancel-safe-futures` crate offers the same as `join_then_try!(file_a.flush(), file_b.flush())?`, plus `join_all_then_try` and `TryStreamExt::{collect_then_try, for_each_concurrent_then_try}`. It is a reasonable dependency when a crate has several of these; the plain `join!` form needs no dependency. The `try_` forms remain right for independent reads, where the rest is useless after one failure.

### 2.4 Stop tasks cooperatively instead of aborting them

Give the task a stop signal and let it check the signal only where its state is consistent:

```rust
async fn sync_loop(stop: CancellationToken) {
    loop {
        tokio::select! {
            biased;
            _ = stop.cancelled() => return,      // only checked between whole batches
            batch = next_batch() => {
                write(batch).await;
                notify_written().await;          // abort could stop between these two
            }
        }
    }
}
```

`CancellationToken` is the Tokio default (`lifetimes-and-load.md` § 3). Oxide's `cancel_safe_futures::coop_cancel` is the same idea with many cancellers and one receiver, and a `Waiter` the canceller can await until the task is gone.

### 2.5 Merge streams instead of a `select!` loop

A `loop { select! { … } }` drops and recreates its branch futures on every iteration. Turning each source into a stream and merging them keeps every in-progress future inside its stream until it finishes. Tokio's `select!` docs recommend this, with Tyler Mandry's `read_send` example:

```rust
enum Event {
    Sent,
    Packet(Vec<u8>),
}

let sends = stream::unfold((file, tx), |(mut file, tx)| async move {
    read_send(&mut file, &tx).await;              // runs to completion; never dropped midway
    Some((Event::Sent, (file, tx)))
});
let packets = stream::unfold(socket, |mut socket| async move {
    let packet = socket.read_packet().await;
    Some((Event::Packet(packet), socket))
});
let mut events = pin!(sends.merge(packets));      // tokio_stream::StreamExt::merge
while let Some(event) = events.next().await {
    match event {
        Event::Sent => {}
        Event::Packet(packet) => handle(packet),
    }
}
```

The loop body still stops both streams while it runs; keep it short and do not await shared resources there (futurelock, `select-and-tasks.md` § 3).

### 2.6 Move the work into a task

Spawn the cancel-unsafe future and wait on its `JoinHandle`. The handle can be dropped or lose a `select!` without touching the work, and it holds no lock or permit, so it also removes futurelock. If the work should stop when the caller stops, wrap the handle in `tokio_util::task::AbortOnDropHandle` — dropping it aborts the task, like dropping a future:

```rust
let mut upload = AbortOnDropHandle::new(tokio::spawn(upload(file)));
let mut progress = tokio::time::interval(Duration::from_millis(500));
loop {
    tokio::select! {
        res = &mut upload => break Ok(res??),
        _ = progress.tick() => report_progress().await,  // cannot futurelock `upload`
    }
}
```

Without a loop, pass the handle by value to `select!`; that makes it obvious nothing stays unpolled. This costs one allocation per task and needs `Send + 'static`.

---

## 3. Testing cancel safety

A cancel-safety claim in a doc comment is a contract; test it like one. The test cancels the future at a chosen `.await`, then checks the public outcome — every item delivered once and in order, the lock still usable, the resource in a valid state — not private fields.

**Cancel at the first await that waits.** `FutureExt::now_or_never` polls once and drops the future. Make the await you want to test pending (a full channel, a held lock, a paused timer), so the single poll stops there:

```rust
#[tokio::test]
async fn flush_cancelled_while_waiting_loses_no_items() {
    let (tx, mut rx) = mpsc::channel(1);
    let mut outbox = Outbox::default();
    outbox.push(1);
    outbox.push(2);
    outbox.push(3);

    // One poll sends 1, then waits for a slot for 2; now_or_never drops the future there.
    assert!(outbox.flush(&tx).now_or_never().is_none());

    let flush_rest = async move { outbox.flush(&tx).await.unwrap() }; // drops tx when done
    let receive_all = async {
        let mut received = Vec::new();
        while let Some(item) = rx.recv().await {
            received.push(item);
        }
        received
    };
    let ((), received) = tokio::join!(flush_rest, receive_all);
    assert_eq!(received, [1, 2, 3]);
}
```

The pop-then-`send` version from § 1.2 fails this test with `[1, 3]`.

**Cancel at every await.** When a function has several awaits, repeat the case for each: poll N times (`futures::poll!`, or `tokio_test::task::spawn(fut).poll()`), let the outside world make the next await ready between polls, drop, finish with a fresh call, and assert the same outcome. A loop over N in one test is fine when the case stays readable.

**Timeouts with paused time.** `#[tokio::test(start_paused = true)]` (Tokio feature `test-util`) advances the clock as soon as every task is idle, so a 5-second timeout fires at once and deterministically:

```rust
#[tokio::test(start_paused = true)]
async fn reserve_times_out_when_the_channel_stays_full() {
    let (tx, _rx) = mpsc::channel::<u32>(1);
    tx.send(0).await.unwrap(); // fill the only slot
    let started = tokio::time::Instant::now();

    let result = tokio::time::timeout(Duration::from_secs(5), tx.reserve()).await;

    assert!(result.is_err());
    assert_eq!(started.elapsed(), Duration::from_secs(5));
}
```

Flag a cancel-unsafe function that the diff puts under `select!`, `timeout`, or a `try_*` combinator with no such test, and a public method documented as cancel safe with none.

---

## Sources

- https://rfd.shared.oxide.computer/rfd/0400 — Dealing with cancel safety in async Rust (Oxide)
- https://rfd.shared.oxide.computer/rfd/0609 — Futurelock (Oxide)
- https://sunshowers.io/posts/cancelling-async-rust/ — Cancelling async Rust (Rain)
- https://docs.rs/cancel-safe-futures/latest/cancel_safe_futures/ — `cancel-safe-futures` (Oxide)
- https://tmandry.gitlab.io/blog/posts/making-async-reliable/ — Making Async Rust Reliable (Tyler Mandry)
- https://blog.yoshuawuyts.com/async-cancellation-1/ — Async Cancellation I (Yoshua Wuyts; drop guards and halt safety)
- https://docs.rs/tokio/latest/tokio/macro.select.html — `tokio::select!` (cancellation safety, alternatives from the ecosystem)
- https://docs.rs/tokio-util/latest/tokio_util/task/struct.AbortOnDropHandle.html — `AbortOnDropHandle`
- https://tokio.rs/tokio/topics/testing — Unit Testing (Tokio; paused time)
