# Isolation: control and data planes, static stability, sharding, cells

Reference for reviewing whether one failure, one tenant, or one bad input can take down more than its share.

---

## 1. Control plane and data plane

### 1.1 Separate them

The **data plane** is everything on the critical path of each request or message: it must be highly available, fast, and scale with traffic. The **control plane** manages configuration and metadata: lower scale, and it favors consistency. Their requirements conflict, so they should not share a path, process pool, or scaling.

- **Flag:** admin or config writes served by the same path and pool as hot-path traffic.

### 1.2 The data plane keeps running on stale config (static stability)

When the control plane is down, the data plane keeps serving with its last known configuration — it may miss updates, but everything that worked before keeps working. A service can be at best as available as its least available **hard** dependency, and availabilities multiply along a chain (0.9999 × 0.99 ≈ 0.9899).

```rust
// Worse: a refresh failure takes the service down with the control plane
let config = control_plane.fetch_config().await?;

// Better: keep serving from the last good config; report how stale it is
match control_plane.fetch_config().await {
    Ok(config) if config.is_valid() => { self.current.store(Arc::new(config)); /* also persist to disk */ }
    Ok(_) | Err(_) => self.metrics.config_age_seconds.record(self.current.load().age_seconds(), &[]),
}
```

- **Keep the last good copy locally** (on disk too), so restarts survive a control-plane outage, and keep a few older versions in case the newest is corrupt.
- **Validate before accepting**: an empty list, a wildcard, or an unusually large change from the previous version needs a check or escalation (see `change-safety.md` § 2).
- **Flag:** a config, flag, schema-registry, or metadata fetch failure that crashes the service, refuses requests, or clears its cached config; a synchronous control-plane call per request.

### 1.3 Startup must not need the control plane

If every instance pulls its full config or state from a control service at boot, a mass restart overloads that service, and then nothing can start. Read a snapshot from a scalable store at startup and take only small deltas from the control plane.

### 1.4 Let the smaller service set the pace

When callers outnumber a service by 100× or more, a small shift — a bug, outage recovery, a few retries — makes the large fleet act together and drives the small service past its breaking point. Make the small side control the rate: it pushes, or it writes to a scalable store (an object in S3) that the large fleet reads and caches. If the large fleet must call in, add jitter and backoff on the clients, tune call frequency, and add load shedding on the service.

- **Flag:** every pod polling a small config or metadata service directly, especially on startup or reconnect, at a fixed fleet-wide interval with no jitter.

### 1.5 Close the control loop

A controller that pushes desired state must also check that it was applied within a reasonable time and act if not. Monitoring is the part most often missing.

- **Flag:** push-and-forget config or rollout; no "applied version" metric or reconciliation.

---

## 2. Static stability across zones

### 2.1 Pre-provision for losing a zone

Recovering by launching new instances depends on the control plane and every bootstrap system (config download, discovery registration, credentials) at the moment they are most likely to misbehave. Overprovision instead: with three zones, run each at about 66 % of its load-tested capacity, so two zones carry everything without launching anything.

- **Flag:** replica counts or autoscaler minimums sized to steady state with "autoscaling will handle a zone loss"; "replace the node" as the failover plan.

### 2.2 Keep critical call paths zone-local

With partial (gray) failures, a request that picks a random zone at every hop avoids a bad zone with probability (2/3)^N over N hops; staying in one zone keeps it at 2/3.

---

## 3. Fault isolation

### 3.1 Without sharding, a poison request takes everyone down

When any worker can serve any request, a request that crashes workers — or a flood — takes out the first worker, then cascades as the others pick up its load. Redundancy does not help, because the cause is in the request, not the machine. Same for a noisy neighbor: one tenant's heavy requests degrade everyone.

### 3.2 Simple sharding limits impact to one shard

Split workers into shards and assign each tenant to one. With four shards, a problem tenant affects 25 % instead of 100 %, at the cost of more spare capacity per shard.

### 3.3 Shuffle sharding limits it to one combination

Give each tenant a random but **permanent** combination of workers. With 8 workers and 2 per tenant there are C(8,2) = 28 combinations, so a problem tenant fully affects only 1/28 of tenants, and any other tenant shares at most one worker with it. It needs:

- clients that retry on the other workers of their shard;
- health-based removal of bad workers;
- a bound on how many workers any two tenants share.

- **Flag:** tenant-to-worker assignment random per request (not permanent); fixed disjoint groups where combinations are possible; no overlap check.

### 3.4 Cells: whole copies of the stack

A cell is a complete, independent copy of the workload — its own load balancer, compute, and storage — serving a subset of tenants chosen by a partition key that matches the service's natural grain. A thin router sends each request to its cell and does nothing else. A bad deploy or poison input then affects one cell. Give each cell a maximum size, load test it at that size, track utilization against it, and grow by adding cells.

- **Flag:** "cells" that share a database, queue, cache, or global config; cells with no tested limit.

### 3.5 Isolate bad sources in messaging

Tag messages with their source (tenant, device). When one source's messages keep failing, move them to a lower-priority channel instead of letting their retries consume everyone's capacity.

### 3.6 Failure domains must not share a change path

Zones or cells that all read one global config, flag, or schema can all be broken by one edit. Partition dependencies along the same lines, and roll changes out one domain at a time (`change-safety.md` § 3).

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/static-stability-using-availability-zones/ — Static stability using Availability Zones
- https://aws.amazon.com/builders-library/avoiding-overload-in-distributed-systems-by-putting-the-smaller-service-in-control/ — Avoiding overload in distributed systems by putting the smaller service in control
- https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/ — Workload isolation using shuffle-sharding
- https://aws.amazon.com/builders-library/reliability-and-constant-work/ — Reliability, constant work, and a good cup of coffee
