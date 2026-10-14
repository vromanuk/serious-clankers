# Downstream: timeouts, retries, idempotency, circuit breakers

Reference for reviewing how a service calls things it depends on. The pass/fail floor lives in hard rules (`HR-io-timeout`, `HR-retry-backoff`); this file covers the design above it.

---

## 1. Timeouts

### 1.1 Every call that might never return gets a timeout

Set a **connect** and a **request** timeout on every remote call — including calls between processes on the same host — and on waits that can block forever (lock acquisition, semaphore permits, blocking receives). Never rely on a client's default: many are infinite.

```rust
// Worse: library defaults, possibly no timeout at all
let client = reqwest::Client::new();

// Better: built once, with bounds, then shared
let client = reqwest::Client::builder()
    .connect_timeout(Duration::from_secs(2))
    .timeout(Duration::from_millis(1_500)) // from the dependency's measured p99.9
    .build()?;
```

- **Flag:** clients built with defaults; only one of connect/request set; a third-party SDK that makes network calls and exposes no timeout setting.
- **Check coverage:** a socket read timeout is not an end-to-end bound, and some clients leave DNS or the TLS handshake uncovered. Prefer the built-in timeouts of well-tested clients.

### 1.2 Derive the value from measured latency

Choose an acceptable false-timeout rate, then use the matching latency percentile of the dependency: 0.1 % false timeouts → its p99.9. Add padding when p99.9 sits close to p50 or the network path is long. Too high wastes resources while waiting; too low triggers retries that add load and can turn a small slowdown into an outage.

- **Flag:** round numbers (30 s, 10 s, 5 s) with no latency data; one shared timeout for dependencies with very different latency.

### 1.3 Propagate deadlines

Pass the caller's remaining time budget downstream (gRPC deadlines do this). A fixed per-hop timeout that ignores the budget keeps working after the original caller has given up.

### 1.4 A timeout means "unknown", not "failed"

After a timeout the client cannot know whether the request took effect — the server may have applied it and crashed before replying. Treat it as UNKNOWN: re-run only idempotent operations (§ 3), and do not commit progress as if it succeeded.

- **Flag:** a timeout handler that assumes "not applied" and repeats a non-idempotent write, or assumes "applied" and moves on.

### 1.5 Timeouts are temporary failures

Classify timeouts and other transient errors as transient. Caching them, treating them as permanent, sending the record to a dead-letter queue, or skipping it, turns a blip into data loss or a wide outage.

### 1.6 One wrapper for every outbound call

Route outbound calls through one shared client, wrapper, or middleware layer that applies the timeout, retry policy, and instrumentation, so no call site can forget. Record, per call: duration, outcome/status, whether a timeout fired, the resource (table, bucket, topic), and the retry count.

---

## 2. Retries

### 2.1 The floor: bounded, capped exponential backoff, jitter

`delay = random(0, min(cap, base × 2^attempt))`, stopped by an attempt cap or an overall deadline. This is `HR-retry-backoff`; flag it there.

### 2.2 Retry only what can succeed later

Retry transient server-side faults. Do not retry client errors (validation, 400/401/403/404) — they fail the same way again. Treat rate-limited responses (429, `RESOURCE_EXHAUSTED`) specially: honour `Retry-After` and slow down.

### 2.3 Retry at one layer

Retries multiply across layers: five layers with three attempts each turn one failing query into 3⁵ = 243 attempts. Retry at one level of a call chain and fail fast everywhere else.

- **Flag:** an application retry loop around an SDK or gRPC channel that already retries; a middle service retrying on behalf of a caller that also retries; mesh/sidecar retries plus code retries on the same call.

### 2.4 A per-request cap does not stop amplification — use a retry budget

Each caller retrying three times is still 3× load on a failing dependency, in aggregate. A **retry budget** shared across requests ties retries to observed health: a token bucket where each success deposits a token and each retry spends several. When the dependency is healthy, retries are allowed; when errors rise, the bucket drains and retries stop.

```rust
/// Retries allowed only while recent calls mostly succeed.
/// With RETRY_COST = 5 tokens and 1 token per success, retries stop
/// once more than roughly 1 call in 6 fails.
struct RetryBudget { tokens: AtomicU32 }

impl RetryBudget {
    fn record_success(&self) { /* add 1, up to MAX_TOKENS */ }
    fn try_spend_for_retry(&self) -> bool { /* subtract RETRY_COST if available */ }
}
```

- **Why not a time-based limit ("50 retries/s")?** It keeps sending 50/s for the whole outage; it does not follow the dependency's health.
- **Why not only a circuit breaker?** A breaker is all-or-nothing; a budget adjusts smoothly (§ 4).
- **Flag:** retries bounded only per request on a hot path called by many concurrent requests.

### 2.5 Batch paths: park, don't wait

In batch or stream processing without real-time needs, put failed work on a retry queue instead of sleeping through long backoffs inline while other work waits behind it.

### 2.6 Make retries visible

Count retry attempts and alert when retries become frequent: retries can sit idle for months and then kick in during a scenario nobody tested.

### 2.7 When the dependency throttles you

Synchronous callers should stop retrying when rate-exceeded responses are frequent. Asynchronous callers (consumers, batch jobs) should apply backpressure: slow their own processing until requests succeed again, instead of hammering at full speed.

---

## 3. Idempotency

### 3.1 Side-effecting calls are retried only if idempotent

A timeout does not mean the side effect did not happen. Create, append, increment, publish, and send operations need idempotency before any retry or redelivery.

### 3.2 Use a caller-provided request id

The caller sends a unique id (a UUID, or a key derived from the input such as topic/partition/offset) and **reuses it on every retry**. The server treats a repeat id from the same caller as a duplicate. Hashing request fields guesses intent and fails when a caller really wants two identical resources.

```rust
// Worse: a fresh random name on each attempt — a retry stores a second copy
let key = format!("{prefix}/{}.csv", Uuid::new_v4());   // inside the retry loop

// Better: the name is chosen once per unit of work and kept by every retry,
// so a retry overwrites its own object
let key = file.key.clone();                            // chosen when the file was opened
```

### 3.3 Record the id atomically with the effect

Store the id and apply every mutation for the request in one atomic operation (one transaction). Otherwise the server can record the id and lose the work — every retry becomes a no-op — or do the work twice.

- **Flag:** "insert key, then do the work" in two steps; a dedup key in a cache or separate table outside the main write's transaction; a handler with external side effects that claims idempotency from a local transaction alone.

### 3.4 Return the original answer for a duplicate

A duplicate gets the same response as the first success, not "already exists" (which changes the caller's control flow and makes default retries unsafe). If the same id arrives with *different* parameters, return a validation error — that is a different intent.

### 3.5 Keep ids long enough, not forever

Retain idempotency records for at least the longest retry or redelivery window of the producers (for example, resource lifetime plus a late-arrival margin), then purge them.

---

## 4. Circuit breakers

### 4.1 When to use one

Retries help when the next call is expected to succeed; a circuit breaker helps when it is expected to fail — a dependency that can stay down for a long time. The fastest call is the one you do not make.

### 4.2 Build it properly or not at all

Trip when failures **including timeouts** exceed a threshold within a time window; while open, fail immediately; after a wait, let a probe through (half-open). Take thresholds and wait times from past failure data, and test every mode — breakers are modal behavior that is hard to test.

- **Flag:** a breaker that ignores timeouts; counters with no time window; no half-open state; constants with no justification; a new breaker with no tests of open and half-open.

### 4.3 Decide what an open breaker returns

Failure of a non-critical dependency should degrade the response (a page without recommendations), not fail it. Write that branch.

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/ — Timeouts, retries, and backoff with jitter
- https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/ — Making retries safe with idempotent APIs
- https://aws.amazon.com/builders-library/challenges-with-distributed-systems/ — Challenges with distributed systems
- https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/ — Avoiding fallback in distributed systems
- https://aws.amazon.com/builders-library/fairness-in-multi-tenant-systems/ — Fairness in multi-tenant systems
- System Design for Interviews and Beyond (Mikhail Smarshchok), reliability module — retry amplification and success-funded retry budgets
