# State and messaging: caches, consumers, partitions, leases

Reference for reviewing state that is cached, consumed from a log or queue, partitioned, or guarded by a leader.

---

## 1. Caches

### 1.1 Caches have modes

A cache makes things faster — until it is empty (cold start, flush, deploy) or overwhelmed, and then the source it protects takes the full load. A service whose dependencies were scaled down because the cache hid the load has become *addicted* to its cache: a cold or failed cache then floods the dependency and takes both down.

- **Rule:** the origin must survive the load with the cache empty or gone — slower is fine, down is not. Load test at a 0 % hit rate.
- **Flag:** a cache added and downstream capacity reduced, with no test at 0 % hits and no cap on miss traffic to the origin.

### 1.2 Local (in-process) caches

Three costs to account for: copies disagree across the fleet; downstream load grows with fleet size; and every new or restarted host starts empty, so a rolling deploy or scale-out sends a burst to the origin.

### 1.3 When the external cache is down

Do not send every miss straight to the origin. Fall back to a small in-process cache, or cap the rate to the origin (load shedding).

```rust
// Worse: a cache outage becomes an origin outage
let value = match cache.get(&key).await {
    Ok(Some(value)) => value,
    _ => origin.get(&key).await?,                  // unthrottled, every request
};

// Better: one fetch per key at a time, and a cap on what reaches the origin
let value = match cache.get(&key).await {
    Ok(Some(value)) => value,
    _ => match local.get(&key) {
        Some(value) => value,                      // stale is better than an error
        None => self.origin_limiter.run(|| origin.get(&key)).await?, // sheds above the cap
    },
};
```

### 1.4 Coalesce concurrent misses

Many requests for the same uncached item at once (a hot key, a host joining with an empty cache) cause a thundering herd. Keep at most one outstanding origin fetch per key (single-flight), or use the cache library's loader.

### 1.5 Soft TTL, hard TTL, and errors

- Refresh after a **soft** TTL, but keep serving the cached value until a **hard** TTL when the origin is unavailable: stale data usually beats an error, and the hard TTL bounds how stale it gets.
- Cache *something* on errors too — the last good value, or a negative entry with its own short TTL — so repeated requests do not hammer a failing or missing resource.
- Choose size and TTLs from modeled traffic and staleness tolerance, and emit hit/miss, size, and origin-call metrics to check them. Add per-host jitter to TTLs so entries do not expire fleet-wide at once.
- Bound every cache and give it an eviction policy. A `HashMap` with no limit is a leak, not a cache.

### 1.6 Cached data is persistent data

Entries in an external cache outlive deploys, so mixed old and new versions read each other's entries. Version the format, read older versions, tolerate unknown fields, and never panic on an unexpected shape (that makes poison pills); discarding mismatched entries fleet-wide can trigger a refresh storm.

---

## 2. Consumers (logs and queues)

### 2.1 Make it durable, then say it is done

Commit the offset, acknowledge, or publish "done" only **after** the side effects (storage writes, downstream calls) are durable. Committing first loses the message if the consumer crashes; committing after risks a duplicate, which idempotency handles.

```rust
// Worse: progress recorded before the work is durable
consumer.commit(&record)?;
store.put(&key, &rows).await?;

// Better: durable first, then the offset
store.put(&key, &rows).await?;
consumer.commit(&record)?;     // or acknowledge, and let the offset tracker commit
```

- **Flag:** auto-commit of offsets on receipt; a commit or ack before the side effects complete.

### 2.2 Consumers must be idempotent

There is no exactly-once delivery; the best a consumer can do is exactly-once *processing*: idempotent handlers plus commit-after-processing. Deduplicate by message id or a key derived from the input (`downstream.md` § 3).

- **Flag:** appends, increments, or creates keyed by nothing stable; object names randomized per attempt.

### 2.3 Finish within the lease

Processing — including remote calls and their retries — must finish well within the visibility timeout, lease, or poll/session interval, or another consumer gets the same message while the first is still working. Under overload this repeats until the service effectively fork-bombs itself. Stop work when the deadline passes, or extend the lease while working.

### 2.4 Cap redeliveries; dead-letter, never drop

A message that always fails is otherwise retried forever, wasting capacity and blocking healthy messages. Cap deliveries, then move the message to a dead-letter channel before removing it. Never "log and skip": deleting an unprocessed message is data loss. Alarm on any message in the dead-letter channel (it means a bug), and keep a way to move messages back after the fix.

- **Flag:** infinite retry of one message (blocking its partition); committing past a failed message; a dead-letter queue with no alarm or redrive.

### 2.5 Every path ends the message

On every path, including errors and panics, the message is acknowledged, nacked, or moved. A forgotten message stays in flight until its timeout.

### 2.6 Ordering and partitions

- Brokers like Kafka order only within a partition, and end-to-end order requires one consumer per partition. Do not process one partition's messages in parallel while assuming order, and check what each broker actually guarantees (ordering, delivery, durability, message size limits).
- **Hot partitions:** pick keys so no partition gets more than one consumer can handle. Low-cardinality or skewed keys (one big tenant, a constant, a date) send everything to one place.
- Choose a fixed partition count deliberately, with headroom: too many adds overhead, too few caps scale, and changing it later reshuffles keys.
- Where nodes or partitions are added and removed (caches, storage shards), use consistent hashing, not `hash % n`, which moves most keys on every change.

### 2.7 Work that must happen is not fire-and-forget

Work the caller has been told is accepted must sit in a durable channel until processed. A spawned task or async call with no durable record is lost if the process dies.

### 2.8 No dual writes

Writing to a database and then publishing to a broker as two steps leaves them inconsistent after a crash in between. Append the message to an outbox table in the same transaction as the change, relay it, and deduplicate at the receiver. Multi-step workflows (sagas) checkpoint state durably on each transition, and their participants deduplicate.

### 2.9 Measure backlog by age

Emit the age of the oldest or average message at first read, from the broker's timestamp. Details and capacity planning: `upstream.md` § 4.

---

## 3. Leader election and leases

- **Use a well-tested lease or lock client** (an established coordination service or its client library); correct leases are subtle.
- **Leases depend on local elapsed time only** — monotonic durations, never wall-clock timestamps compared across hosts.
- **Check the lease immediately before every side effect** outside the leader, and subtract a safety margin: network delay, retries, and pauses can expire it early.
- **Do not renew the lease from a detached background task** that cannot stop the work when renewal fails, or that keeps the lease alive after the worker died.
- **Design for zero or two leaders.** Elections give "mostly one". Make leader work idempotent and protect writes with a version check or fencing token in the store.

- **Flag:** a hand-rolled lease with a timestamp column and `WHERE expires_at < now()`; one leadership check at the top of a long batch of external writes; unconditional leader-only writes.

---

## 4. Failure detection

No failure detector is perfect: a timeout cannot tell slow, crashed, and unreachable apart. Use heartbeats only where a peer's loss must be acted on quickly; otherwise detect failure on the next call. After marking a peer down, keep probing so recovery is noticed.

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/caching-challenges-and-strategies/ — Caching challenges and strategies
- https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/ — Avoiding insurmountable queue backlogs
- https://aws.amazon.com/builders-library/leader-election-in-distributed-systems/ — Leader election in distributed systems
- https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/ — Making retries safe with idempotent APIs
