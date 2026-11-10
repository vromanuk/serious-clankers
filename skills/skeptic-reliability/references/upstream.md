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

### 5.6 Considerations: fixed slots and passes over every slot

These are ideas to weigh, not review rules. They fit a component with a hard maximum and a latency budget (an order-matching engine, a session table, a connection pool); they do not fit most request handlers. The running example is an order book that must hold at most one million open orders.

**Allocate every slot at startup, and reject at the limit.** The usual design allocates an order when it arrives and frees it when it completes. That brings uncertainty a latency-sensitive engine may not want: allocation takes unpredictable time, memory fragments, allocation can fail under load, and threads compete inside the allocator. The alternative is to allocate one million slots at startup: a new order takes a free slot, a finished order frees it, and a full book rejects the next order. TigerBeetle goes further — no allocation at all after startup ("TigerStyle").

Why reject when memory is still free? Because the limit is what was designed and tested. Growing past it trades one extra order for the risk that the kernel's OOM killer ends the process and loses the million orders already held. Rejecting one request is the smaller failure, and it is the same choice as load shedding (§ 1.1).

What it costs:
- the memory is held even when the book is empty;
- the maximum must be a stated number — Little's Law gives one: open orders = arrival rate × how long an order stays open;
- "book full" becomes a normal answer that callers must handle and that needs its own metric;
- tests must run at the maximum (§ 5.4), or the full case is never exercised.

**Visit every slot, not a list of active ones.** The usual way to manage the slots keeps a second collection with the indexes of active orders and loops over it: 100 orders cost 100 steps, 100 000 cost 100 000. The constant-work alternative treats every slot as always holding an order in some state, with "reserved" as the empty state, and each pass visits all of them:

```rust
/// One preallocated place for an order. `Reserved` is an empty slot: it holds no stale
/// order data, and every slot is the same size whichever variant it holds.
enum Slot {
    Reserved,
    Active { side: Side, order: Order },
}

/// Open orders in a fixed number of slots, allocated once at startup.
pub struct OrderBook {
    slots: Box<[Slot]>,
    /// Indexes of reserved slots. Allocated at startup with room for every slot, so it never grows.
    free_slots: Vec<usize>,
}

impl OrderBook {
    pub fn new(max_open_orders: usize) -> OrderBook {
        OrderBook {
            slots: vec![Slot::Reserved; max_open_orders].into_boxed_slice(),
            free_slots: (0..max_open_orders).rev().collect(),
        }
    }

    /// Puts the order in a reserved slot, or rejects it when every slot holds an open order.
    pub fn place(&mut self, side: Side, order: Order) -> Result<SlotIndex, BookFull> {
        let index = self.free_slots.pop().ok_or(BookFull)?;
        self.slots[index] = Slot::Active { side, order };
        Ok(SlotIndex(index))
    }

    /// Returns the slot to reserved and gives back the order, if the slot still holds order
    /// `id`. A slot is reused after removal, so a stale index must not remove a newer order.
    pub fn remove(&mut self, slot: SlotIndex, id: OrderId) -> Option<Order> {
        match self.slots[slot.0] {
            Slot::Active { order, .. } if order.id == id => {
                self.slots[slot.0] = Slot::Reserved;
                self.free_slots.push(slot.0);
                Some(order)
            }
            Slot::Active { .. } | Slot::Reserved => None,
        }
    }

    /// Visits every slot, open or not, so the cost is the same at any load.
    pub fn sum_open_shares(&self, side: Side) -> u64 {
        self.slots
            .iter()
            .map(|slot| match slot {
                Slot::Active { side: slot_side, order } if *slot_side == side => {
                    u64::from(order.shares)
                }
                Slot::Active { .. } | Slot::Reserved => 0,
            })
            .sum()
    }
}
```

Doing more iterations can still be the better trade:
- **Memory locality:** the active-index loop jumps around memory; the full pass reads it in order. CPUs fetch memory in cache lines (64 bytes) and prefetch the next ones when access is sequential.
- **Simpler code for the compiler:** a plain loop over contiguous memory is easier to optimize (no index lookup, branches that predict well).
- **Predictable cost:** a pass costs the same with ten orders or a million, so a load test measures the worst case, and nothing gets slower at the moment the book fills up.
- **Less bookkeeping:** there is no active list to keep in sync with the slots.

Things to weigh before choosing it:
- **The pass costs the maximum every time, even when idle.** That is the point for latency, but it wastes CPU when the book is mostly empty and work is billed or shared.
- **Size the pass in bytes.** Here `Order` is 32 bytes and `Slot` is 48: a `u128` id aligns the whole slot to 16 bytes, so the one-byte `side` and the variant tag cost a full 16. One million slots is about 48 MB per pass, much larger than CPU caches, so a pass is limited by memory bandwidth — at a rough 10–20 GB/s, a few milliseconds. Measure it against the latency budget. Storing the id as `[u64; 2]` makes the slot 32 bytes (32 MB per pass); splitting the fields a pass reads into their own array cuts the bytes further.
- **Finding a free slot still needs bookkeeping.** The free list above is allocated once and never grows, so placing stays O(1) without a list of active orders. Scanning for a reserved slot instead makes placing cost a full pass.
- **Slot indexes are reused.** A caller holding an old index can reach a newer order; `remove` checks the order id for that reason (a generation counter per slot is the other common answer).
- **In Rust, an enum of slot states keeps the empty state honest.** A struct with a `state` field leaves price and shares readable in a reserved slot; `enum Slot` has no stale fields, and every slot is still the same size.

Related ideas: batching (`skeptic-async/references/select-and-tasks.md` § 2.7) makes work grow slower than load; constant work makes it not grow at all. Both remove the moment where a busier system takes a different, less-tested path.

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/ — Using load shedding to avoid overload
- https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/ — Avoiding insurmountable queue backlogs
- https://aws.amazon.com/builders-library/fairness-in-multi-tenant-systems/ — Fairness in multi-tenant systems
- https://aws.amazon.com/builders-library/reliability-and-constant-work/ — Reliability, constant work, and a good cup of coffee
- https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md — TigerStyle (TigerBeetle; static allocation at startup)
- System Design for Interviews and Beyond (Mikhail Smarshchok), reliability module — useful vs total throughput, deadline-aware and priority shedding
