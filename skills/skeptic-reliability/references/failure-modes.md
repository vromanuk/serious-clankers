# Failure modes, redundancy, and fallbacks

Reference for asking "what happens when this fails?" about each part a change adds or touches. Rust examples; the reasoning applies to any service.

---

## 1. How to review

### 1.1 Walk every component and dependency

For each new component, dependency, or remote call, state what happens when it is slow, down, wrong, or returns garbage. Then remove the single point of failure with redundancy, or, when that is impossible, shrink how much it can break (its blast radius).

- **Flag:** a single instance on the request or processing path; a new hard dependency with no stated failure behavior.
- **Ask:** "What does the caller see when this dependency is down for ten minutes? For a day?"

### 1.2 Weight findings by likelihood × impact

Not every possible fault deserves handling. Score each on likelihood (1–3) and impact (1–3) and multiply: 1–2 is low, 3–4 medium, 6–9 high. Fix likely, high-impact faults first.

- **Flag:** elaborate handling for an unlikely, low-impact fault while a likely one (a dependency timing out) has none.

### 1.3 More parts, more failures

Splitting into services, partitions, and replicas adds moving parts, and every part can fail. A change that adds parts must say how each one's failure is handled.

---

## 2. Common causes of failure

### 2.1 Error handling

Most catastrophic failures in distributed data stores came from mishandled *non-fatal* errors: errors ignored, a generic catch that killed the process, or half-written handlers. Simple tests would have caught most of them.

```rust
// Worse: the error disappears, and so does the record
let _ = store.put(&key, &bytes).await;
match store.put(&key, &bytes).await.ok() { /* … */ }

// Better: every outcome has a decision, and the decision is tested
match store.put(&key, &bytes).await {
    Ok(()) => acknowledge(record),
    Err(error) if error.is_retryable() => retry_later(record, error),
    Err(error) => park_in_dead_letter(record, error), // never silently drop
}
```

- **Flag:** `let _ =` / `.ok()` / an empty `Err(_) => {}` on a fallible remote call; `panic!`/`process::exit` on a recoverable error; `TODO` inside an error branch; new error branches with no test.

### 2.2 Classify every remote-call outcome

A remote call has more than two outcomes. Handle at least: never sent (connection refused — the server cannot have it), retryable, fatal, **unknown** (timed out — it may or may not have happened), and success with an invalid or corrupt reply. Each needs different handling.

- **Flag:** one `Err(_) => retry` or `Err(_) => fail` branch for every error kind.
- **Test:** each outcome of each call, and sequences where an earlier call failed or timed out. Mocks that never time out hide the hardest cases.

### 2.3 A fast, empty "success" is a failure

A server that hits an error (a full disk) and answers with fast, empty successful responses can attract most of the traffic from a latency-based balancer and take the whole service down.

- **Flag:** an error branch that returns an empty result with a success status; least-latency balancing with no error or response-size signal.

### 2.4 Configuration changes

Config changes are a leading cause of outages — including *valid* changes that switch on rarely used code that no longer works, or never did. Details in `change-safety.md` § 2.

### 2.5 Single points of failure people forget

DNS (an expired domain, a resolver or top-level-domain outage) and TLS certificates (expiry) are classic single points of failure. So is a backend that lives in one datacenter behind a multi-region service.

- **Flag:** every region pointing at one regional store, bucket, broker, or secret store; certificates with no renewal automation or expiry alert.
- **Startup dependencies:** list every service needed to *start*, including DNS and the local clock, and what happens when each is missing. Hidden startup dependencies decide whether a service can come back after an outage.

### 2.6 Resource leaks

- **Hung calls drain pools.** A call with no timeout, made while holding a pooled thread or connection, never returns it. Async does not help: the socket is never returned to the connection pool. Bounded pools run dry and the process stalls. (`HR-io-timeout` is the pass/fail floor.)
- **Collections that only grow.** A map keyed by request, tenant, or partition with no removal path is a memory leak; leaks lead to swapping, slowness, then allocation failures.

### 2.7 Load pressure and cascading failure

- **Survivors must carry the load.** When one of two replicas drops out, the other serves twice its traffic. If it cannot, timeouts and retries pile on, it fails too, and the failure cascades.
- **Identical limits fail together.** Every host runs the same code with the same limits (file descriptors, pool sizes, memory caps); evenly balanced load makes them all hit the limit at once. A limit whose exhaustion crashes the process is worse than one that sheds work.
- **Reject, don't crash.** Under overload, serve errors for the excess instead of accepting everything and dying: a crash removes *all* of a component's capacity and pushes its load onto peers.
- **Keep a big lever.** Timeouts cause retries, which add load that moves from replica to replica, and the loop can outlive the original fault. Operators need a way to block or sharply cut traffic.
- **Flag:** two replicas each above 50 % utilization; capacity plans that assume every replica is healthy; no kill switch or admission control.

### 2.8 Human and automated operations

Anything that acts on many servers at once — an operator, a script, a controller — can cause a correlated failure. Multi-step manual procedures done in a fixed order will eventually be done wrong.

- **Flag:** fleet-wide parallel commands with no batch size or stop condition; runbooks that need N commands typed in order; manual failover.

---

## 3. Redundancy

### 3.1 Redundancy must earn its complexity

Redundancy helps only when all four hold:

1. It adds more availability than its own complexity costs.
2. Healthy and unhealthy copies are detected reliably.
3. The system can run degraded (survivors have capacity).
4. The system can return to full redundancy (failed copies get replaced).

- **Flag:** failover machinery more complex than the component it protects; removed nodes never replaced; "redundant" copies with no health detection.

### 3.2 Correlated failures defeat redundancy

N copies only multiply availability if they fail independently. List what all copies share: zone, rack, config source, DNS name, binary version, credentials, an operator action. A backup in the same account, region, and credential scope as the primary shares its root cause.

- **Flag:** "N replicas" as the availability argument while every replica shares one zone, config, or dependency; failure domains (zones, cells) that read one global config or flag, so one edit reaches them all.

### 3.3 Spread time-based behavior

Many hosts changing behavior at the same moment creates a correlated spike. Add per-host variation to cache TTLs, token refresh, housekeeping schedules, and "reload everything" triggers. Seed the variation with a stable host value (name, IP) so each host's behavior stays reproducible.

---

## 4. Fallbacks and degraded modes

### 4.1 Prefer making the primary path reliable

A fallback path runs rarely, so it carries latent bugs, and it usually trades something away (slower, weaker, staler) at the worst moment. Investing in the primary path usually beats an alternate one. Often the best fallback is to return the error and let the caller retry.

- **Flag:** a second data source, client, or "degraded mode" that only runs in an error branch.

### 4.2 If an alternate path is essential, run it all the time

Run both paths continuously in production and treat both results as equally valid. Then it is failover, not fallback, and its bugs show up early. An alternate path that quietly serves normal load is the opposite problem: it gets overloaded in the outage it exists for.

### 4.3 Never fall back from a protective layer to what it protects

When a cache or queue in front of a store fails, sending all traffic straight to the store turns a partial outage into a full one. Cap the rate to the store or shed load. Details in `state-and-messaging.md` § 1.

### 4.4 Degraded mode must survive a restart

A flag that disables a dependency at runtime does not help if startup still needs that dependency: the next restart takes the service down.

- **Flag:** a fallback that skips a dependency at runtime while `main` still blocks on connecting to it, fetching its config, or loading its keys.

### 4.5 Degraded output must say so

Mark data served from a cache or fallback as possibly stale (a timestamp or version), turn off features that are no longer safe, and record that the degraded mode is active — a counter or gauge per mode, with its level.

### 4.6 Fallbacks must not weaken security

Attackers can push a system onto its weaker path on purpose. A fallback must enforce the same authorization, TLS verification, and validation as the primary. → security stage.

### 4.7 A buffer for outages must be durable

A queue added to ride out a downstream outage brings its own failure modes. Acknowledging work (committing an offset, returning 200) while it sits only in memory loses it on a crash.

### 4.8 Say which way each failure fails

For every dependency failure, state whether the component fails open (keeps serving — availability) or closed (refuses — safety), and why. Leaving it to a library default is not a decision. Security checks fail closed (→ security stage); most data-plane dependencies fail static (keep the last good state) — see `isolation.md` § 1.

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/challenges-with-distributed-systems/ — Challenges with distributed systems
- https://aws.amazon.com/builders-library/minimizing-correlated-failures-in-distributed-systems/ — Minimizing correlated failures in distributed systems
- https://aws.amazon.com/builders-library/avoiding-fallback-in-distributed-systems/ — Avoiding fallback in distributed systems
- System Design for Interviews and Beyond (Mikhail Smarshchok), reliability module
