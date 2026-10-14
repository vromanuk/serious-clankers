---
name: skeptic-reliability
description: >
  Reliability review: what happens when each part fails — failure modes and
  redundancy, timeouts, retries and idempotency, load shedding, rate limits and
  backlogs, constant work, control vs data plane and static stability, shuffle
  sharding and cells, caches and consumers, rollback and rollout safety, health
  checks, and recovery. Use when the user asks for a reliability review, or when
  skeptic stage 2 (architecture) runs and the diff touches remote calls, queues,
  caches, config, deployment, or state that must survive failure.
---

# Skeptic reliability

**Question:** When a dependency is slow or down, load spikes, a tenant misbehaves, a change goes bad, or data is lost — does this design contain the damage and recover, or does the failure spread?

## When it runs

Run when the diff touches any of: remote calls (HTTP, gRPC, database, broker, object storage), retries, consumers or producers, queues, caches, configuration or feature flags, deployment or health checks, startup dependencies, shared resources between tenants, or state that must survive a crash. Otherwise report `reliability: none` with a one-line reason.

## Load

Load only what the diff touches:

1. `references/failure-modes.md` — failure walk, risk score, common failure causes, redundancy and correlation, fallbacks and degraded modes  
2. `references/downstream.md` — timeouts, unknown outcomes, retries (one layer, retry budgets), idempotency keys, circuit breakers  
3. `references/upstream.md` — load shedding, load leveling, rate limits and quotas, backlogs, constant work, bounded work per request  
4. `references/isolation.md` — control vs data plane, static stability, smaller service in control, zones, sharding, shuffle sharding, cells  
5. `references/state-and-messaging.md` — caches and their modes, consumers (commit after durable, idempotent, leases, dead letters), partitions, leases  
6. `references/change-safety.md` — rollback-safe formats (prepare/activate), config as code, staged rollout, health checks  
7. `references/recovery.md` — known good state, time and clocks, credential rotation and revocation, backups and restores, failover  

## Do

1. **Failure walk:** for each new or changed component, dependency, and remote call, state what happens when it is slow, down, or returns garbage. Flag single points of failure (including DNS, certificates, single-homed backends, startup dependencies) and unstated failure behavior. Weight each finding by likelihood × impact.  
2. **Error outcomes:** every remote call distinguishes never-sent, retryable, fatal, unknown (timeout), and corrupt reply; no error is swallowed; a timeout is never treated as "not applied" for a non-idempotent operation.  
3. **Downstream:** timeouts derived from the dependency's measured latency and covering connect, DNS, and TLS; deadlines propagated; retries only for transient errors, at one layer, with a shared retry budget on hot paths; side-effecting calls retried only with an idempotency key recorded atomically with the effect; breakers built with timeouts counted and a half-open state.  
4. **Upstream:** a concurrency limit with a cheap reject path; work whose caller already gave up is dropped; health checks never shed; thresholds from an overload test; per-tenant, cost-weighted quotas; backlog plan (catch-up capacity, size and age bounds, message-age metric); config and state propagated by constant work rather than per-change events.  
5. **Isolation:** the data plane keeps serving on its last good config when the control plane is down, and can start without it; the smaller fleet sets the pace; capacity survives losing a zone without launching anything; one tenant or poison input cannot reach every worker (sharding, shuffle sharding, cells, low-priority channel for bad sources).  
6. **State and messaging:** the origin survives an empty or failed cache; misses coalesced; soft and hard TTLs; offsets committed only after side effects are durable; consumers idempotent; processing finishes within the lease; redeliveries capped and failures dead-lettered with an alarm, never dropped; partition keys not hot; no dual writes; leases from a tested client, checked before each side effect, with fencing.  
7. **Change safety:** data-format changes split into prepare and activate with all readers verified first; versioned, tolerant readers; upgrade-downgrade tested; config validated at load and rolled out like code; staged by failure domain with automatic rollback, bake time, and capacity kept; health checks shallow for routing, failing open when all hosts fail.  
8. **Recovery:** a way back for every change; durations from a monotonic clock; no time bombs; replays checked against the record's own time; credentials rotate and expire safely; backups verified on restore with checksums covering the key; failover exercised.  
9. **Fallbacks:** flag fallback paths that run only on errors, fall through to the protected store, weaken security, need the failed dependency at startup, or degrade silently.  
10. **Findings:** skeptic concern format with the hazard named (single point of failure, retry amplification, unknown outcome, cascading failure, backlog, poison pill, rollback-unsafe format, …), path:line, what happens at runtime, and the smallest fix. LETTER options for real design forks (e.g. fail open vs fail static, shuffle sharding vs cells).  
11. When in scope: `reliability: ok` (one line naming the failure paths checked) or findings.  

## Do not

- Pass/fail floors → hard rules: missing timeout (`HR-io-timeout`), unbounded or unjittered retries (`HR-retry-backoff`) — reference them, do not repeat them  
- In-process async concerns (cancellation, channel sizing, `Semaphore`, Little's Law for one queue) → `skeptic-async`  
- Alert rules, SLOs, dashboards → observability; here only ask that degraded modes, shedding, retries, and backlog age are measured  
- Authentication, authorization, secrets, attacker-driven load → `skeptic-security` (cross-reference when a reliability choice weakens security)  
- How failure paths are tested → testability and unit-tests; here only whether the failure path exists and can be exercised  
- Demand redundancy, cells, or multi-region for a component whose failure is low-likelihood and low-impact — say why it is acceptable instead  

## Output section title

`### Reliability` inside `## 2. Architecture` when skeptic runs; `## Reliability` when run alone.
