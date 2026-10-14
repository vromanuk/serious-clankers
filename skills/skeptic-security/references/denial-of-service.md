# Denial of service: attacker-driven cost

Reference for reviewing what an attacker — or a buggy client — can make the service spend. Generic overload protection (load shedding, quotas, backlogs) is reliability's (`skeptic-reliability` → `upstream.md`); this file asks what an adversary would aim at.

---

## 1. What attackers aim at

### 1.1 Expensive operations

Sophisticated attackers send the requests that cost the most to answer (search, scans, exports, large aggregations), not simple floods. Bound the cost of each request type and make sure expensive ones cannot be triggered cheaply at volume.

- **Flag:** unbounded page size, time range, or result count; a costly query reachable without authentication or per-caller limits.

### 1.2 Reject early and cheaply

Each layer protects the one behind it, and attack traffic gets more focused and costly the deeper it reaches. Put rate limits, authentication, and size checks before decompression, full deserialization, or database calls.

- **Flag:** a handler that decompresses or fully decodes before checking authentication, size, or quota; no limit on request body or message size; allocation sized by a caller-supplied length.

### 1.3 Amplification

Avoid unauthenticated paths where a small request produces a much larger response, or fan-out to a destination the caller chooses: attackers reflect small requests off open servers toward a victim.

### 1.4 Egress

Repeated requests for a large resource can saturate outbound bandwidth. Cap, rate-limit, or deprioritize large responses and bulk downloads per caller.

### 1.5 Fan-out the caller controls

One inbound request should cause a bounded, roughly constant number of backend calls, including retries. Per-item backend calls in a loop over request data, or fan-out sized by the caller, let one request become thousands.

---

## 2. Surviving it

### 2.1 Serve overload errors, degrade, never crash

When load exceeds capacity, return overload errors or a reduced feature set (read-only, fewer features) rather than queueing without bound or crashing. On resource exhaustion, fail fast and predictably.

### 2.2 Deadlines everywhere

Every request and sub-request has a deadline, propagated downstream, and work is cancelled once it cannot finish in time. One missing deadline is a weakness for the whole system.

### 2.3 Clients must not amplify an attack

Retries with exponential backoff and jitter, at one layer, in a shared client with strict limits — especially for overload errors (429, 503, `RESOURCE_EXHAUSTED`). Hand-written retry loops per call site are easy to get wrong (`HR-retry-backoff`).

### 2.4 Policy dependencies fail static

When the system that supplies rate limits or protection policy is unavailable, keep the last policy. Failing closed blocks all traffic; failing open lets the attack through.

### 2.5 No remote off-switch for the whole fleet

A server decides to shed everything from its own local signals, not only from a remote flag or health signal that an attacker could fake to force the whole fleet into an outage. Automated responses (auto-quarantine, mass shedding) are capped in size and scope; past the cap, a human decides.

### 2.6 Know it is an attack

Monitor request rate, not only CPU and memory: a CPU spike or OOM alone does not show that traffic is the cause.

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
