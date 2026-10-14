# Upstream: overload, load shedding, rate limits, backlogs, constant work

Reference for reviewing how a service protects itself from the load its callers or producers send. In-process queue and channel sizing (Little's Law, bounded channels, `Semaphore`) is the async stage's (`skeptic-async` → `lifetimes-and-load.md`); this file covers the service as a whole.

---

## 1. Load shedding

### 1.1 Reject excess work fast so accepted work finishes in time

Past a tipping point, latency climbs until every request exceeds the client's timeout. Total throughput stays high while **useful** throughput (requests finished before the caller gave up) collapses to zero. A fast "no" (503) beats a slow "yes" that arrives after the caller left.

- **Mechanism:** count in-flight requests (or messages) and reject above a capacity threshold — before parsing, auth lookups, or database calls.

```rust
// Admission control: reject immediately instead of queueing without limit
let Ok(permit) = self.in_flight.clone().try_acquire_owned() else {
    self.metrics.shed.add(1, &[KeyValue::new("operation", op)]);
    return Err(Status::unavailable("overloaded"));     // cheap: no body parse, no log line
};
```

- **Flag:** no concurrency limit; an unbounded executor, queue, or spawn per request; overload that ends in timeouts or OOM instead of fast rejection.

### 1.2 Keep the reject path cheap

An accidental log line, a full body parse, or an auth call in the reject branch can make dropping a request nearly as expensive as serving it. Rejection is never free (TLS, reading part of the request), so shedding alone cannot absorb load that keeps growing — it buys time for scaling.

### 1.3 Choose what to drop

- **Drop work whose caller already left.** Check the remaining deadline when work is dequeued; drop it if the deadline passed.
- **Shed oldest or lowest-priority first.** The oldest requests are the next to time out and be retried. Classify by importance: health checks and auth last, background jobs first. Shedding also needs a notion of request *cost*.
- **Never shed load-balancer health checks**, and reserve capacity for them: a host whose pings time out is removed, shrinking the fleet when capacity is already short.
- **Measure durations with a monotonic clock** (`Instant`, not `SystemTime`): wall clocks jump when NTP syncs.

### 1.4 Set the threshold from an overload test

Load test far past the breaking point (2–10× capacity), measure client-perceived success, and set the limit below the knee where useful throughput starts to fall. Untested overload behavior should be assumed to fail in the worst possible way. Long term, prefer relative signals (CPU, queue wait) over static request rates.

- **Flag:** a shedding or throttling setting with no load-test result beyond saturation; tests that stop at "expected peak".

### 1.5 Don't hide the autoscaling signal

If the shedder rejects at the same metric and level the autoscaler targets (both at CPU 60 %), shedding keeps the metric low and scaling never fires.

### 1.6 Instrument shedding separately

Count rejections by client and operation. Keep shed and failed requests out of the success-latency histogram: fast rejections make the median look great while real latency is terrible.

---

## 2. Load leveling

A queue between callers and a service smooths short spikes when callers do not need a prompt answer. It adds no capacity: if the consumer never catches up on average, the backlog grows forever (§ 4). Pair it with autoscaling, and with rejection for surges scaling cannot absorb.

- **Flag:** a queue added to "fix" sustained over-capacity; no capacity plan behind it.

---

## 3. Rate limiting and quotas

### 3.1 Rate limiting is not overload protection

Load shedding decides from **local** state (requests in flight in this process). Rate limiting decides from **global** state (requests for this key across all instances) and needs coordination. Throttled clients can keep sending, and each rejection still costs a TLS handshake and a partial read, so rate limits only partly help against floods. Use both, for different jobs.

### 3.2 Quotas per tenant, weighted by cost

A sudden rise in total load is usually one tenant (tenant load is mostly uncorrelated). A per-tenant quota rejects only the unplanned part; a global limit fails everyone. Count expensive requests as more than one (a scan is not a get), or debit the true cost after the call.

- **Flag:** a multi-tenant service or shared consumer with only a global limit; one token per call for operations whose cost differs by orders of magnitude; burst capacity sized to minutes of traffic (too large a burst defeats the limit).

### 3.3 Implementation details that matter

- Sliding window over fixed buckets (two counters per key), not a list of timestamps per key.
- Atomic increment or compare-and-swap on shared counters, never read-modify-write.
- Batch counter updates in memory and flush periodically, instead of a remote write per request.
- **Fail open on the limiter store:** if it is unreachable, keep serving from the last known state rather than rejecting traffic.

### 3.4 Tell the client, and make clients listen

Return 429 naming the exhausted quota, by how much, and `Retry-After`. Clients must honour 429 and `Retry-After`; a client that retries 429 immediately turns throttling into a flood. Expose usage-versus-quota per client so callers see they are near the limit.

---

## 4. Backlogs

### 4.1 Queues are bimodal

A queue either has no backlog or a growing one, and the longer a backlog builds, the longer it takes to drain. Causes: more producers, a slower consumer, and failing messages redelivered again and again.

### 4.2 Size consumers for catch-up, not just steady state

After an hour of stopped processing, recovery needs **double** the normal capacity for another hour — and the dependencies the consumer calls must take that catch-up burst too. A one-tenant spike of 10× consumer capacity that runs for 30 minutes before anyone reacts takes 300 minutes to drain.

- **Flag:** consumers sized exactly to the arrival rate; no time-to-drain estimate; catch-up that calls a downstream store with no rate limit; a change that slows per-message work (a new remote call, a longer timeout) with no capacity check.

### 4.3 Bound size and age; measure age

Bound every internal queue by size **and** by how long an item may wait; drop or sideline items that are too old. Emit message age at dequeue (now minus the record's enqueue timestamp), keeping first attempts separate from retries — throughput or lag counts alone do not show how far behind a consumer is.

### 4.4 Keep fresh work flowing

- **Throttle per tenant before work enters a shared queue**, through a thin API that authenticates and tags messages with the tenant, instead of exposing the queue directly.
- **Spread tenants over a fixed set of queues**: hash each tenant to a small subset and enqueue into the least-full one, so a surging tenant backs up only its queues.
- **Sideline over-rate or old messages** to a spillover queue processed after the live one is caught up — an approximation of LIFO that lets fresh data through.
- **Cap concurrency per workload.** By Little's Law, in-flight work = arrival rate × latency: 100 messages/s at 100 ms is 10 in flight, but at 10 s latency it is 1,000. One slow sink must not take every worker.
- **Push back on producers** when a backlog grows, where the business accepts it: scale the inbound throttle down as backlog grows.

### 4.5 Every dequeued message ends somewhere

Every path — including errors and panics — acknowledges, nacks, or moves the message. A forgotten message sits in flight until its visibility timeout, inflating counts and redeliveries. Poison messages go to a dead-letter queue (`state-and-messaging.md` § 2).

---

## 5. Constant work

### 5.1 Make the worst case the same as the average case

Systems change behavior under overload, at the worst moment. A constant-work design does the same amount of work per unit of time whether one thing changed or everything did: no modes, and no work that grows with load.

### 5.2 Propagate full snapshots, not one message per change

Have the producer (a control plane) write the **whole** configuration to a scalable store periodically, and consumers read and apply the whole snapshot on a fixed loop, even when nothing changed. A mass change (quotas updated for every customer) then costs nothing extra, a corrupt snapshot is fixed by the next one, and a bad change is reverted by writing a new snapshot.

- **Flag:** config, routing, or allow-list updates delivered as per-change events whose cost grows with the number of changes; a delta-only stream with no periodic full resync.

### 5.3 No special failure modes

Evaluate state at use time instead of turning a mass event (a zone losing power) into millions of change operations or a special "degraded mode" entered only during incidents. Rarely used modes trigger rare bugs and break operators' mental models. On a data-plane hot path, prefer stores with predictable performance (simple key-value) over ones with hidden query-plan modes.

### 5.4 Pre-allocate for the maximum and test at it

Define the maximum supported size (tenants, config slots, items per snapshot), pre-allocate for it, and load test at that size, so growth never changes the work and tests are representative.

### 5.5 Bounded work per request

Paginate any API that can return a list and cap per-request memory, CPU, and bandwidth: admission control is impossible when the cost of a request is unknown.

- **Flag:** list, query, or scan endpoints with no page size or continuation token; a consumer that loads an unbounded result set into memory.

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/ — Using load shedding to avoid overload
- https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/ — Avoiding insurmountable queue backlogs
- https://aws.amazon.com/builders-library/fairness-in-multi-tenant-systems/ — Fairness in multi-tenant systems
- https://aws.amazon.com/builders-library/reliability-and-constant-work/ — Reliability, constant work, and a good cup of coffee
- System Design for Interviews and Beyond (Mikhail Smarshchok), reliability module — useful vs total throughput, deadline-aware and priority shedding
