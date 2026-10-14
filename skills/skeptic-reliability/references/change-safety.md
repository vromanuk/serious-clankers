# Change safety: rollback, configuration, rollout, health checks

Reference for reviewing whether a change can be deployed and undone without an outage. Most production incidents are self-inflicted by changes.

---

## 1. Rollback safety

### 1.1 A data-format change is a protocol change

The most common reason a rollback is impossible: new code writes data — to storage, a topic, a cache, a file — that old code cannot read, even mid-deployment while both versions run. Treat any change to how data is persisted or exchanged as rollback-unsafe until proven otherwise: serializer, encoding, compression, field rename or removal, type change, and timing contracts peers enforce (heartbeat or session timeouts).

### 1.2 Split it: prepare, activate, clean up

1. **Prepare** — every reader accepts both old and new formats; writers still write the old one.
2. **Activate** — writers switch to the new format.
3. **Clean up** — remove the old reader only after nothing old can still be read.

Each step can be rolled back alone.

- **Before activating**, verify that *every* host (and every consuming service) runs the prepare version — not a minimum-healthy percentage.
- **Bake** between prepare and activate (usually days).
- **Before cleanup**, backfill: data written before prepare — in storage, in a topic within its retention, in caches — stays in the old format until rewritten.
- **Order:** readers deploy before writers when rolling forward; writers roll back before readers.
- **Flag:** one change that both adds the new reader and starts writing the new format; a legacy decoder deleted while old records may still exist.

### 1.3 Version what you write; tolerate what you don't know

Give each serializer change its own version (independent of the build), store it with the data or in metadata, keep older versions readable, and emit metrics of versions read and written. Readers accept unknown fields and, where feasible, keep them on write-back, so old servers do not erase new fields.

```rust
// Worse: a new field from a newer writer makes every old reader fail
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct StoredRecord { id: String, value: f64 }

// Better: versioned, and tolerant of fields this version does not know
#[derive(Deserialize)]
struct StoredRecord {
    #[serde(default)] // records written before versioning read as version 0
    format_version: u16,
    id: String,
    value: f64,
    #[serde(flatten)]
    unknown: serde_json::Map<String, serde_json::Value>, // kept on write-back
}
```

### 1.4 Prove rollback, don't claim it

Run an upgrade-then-downgrade test before production in a multi-instance environment that uses the production deployment settings: deploy to half (old and new together), then all, then roll back the same way, each step long enough for every API and job to run.

### 1.5 Irreversible changes say so

Every change (code, config, schema) has a tested way back to the last known good state, or is explicitly marked irreversible with a plan.

---

## 2. Configuration

### 2.1 Configuration is code

Config, feature flags, rate limits, and infrastructure live in version control and go through the same review, staged rollout, monitoring, and automatic rollback as code. Global, unreviewed config pushes are among the most common causes of outages.

- **Flag:** config edited in a console, by a direct ConfigMap edit, or by an environment override outside version control; a flag flipped fleet-wide in one step; production config pointing at test endpoints.

### 2.2 Validate when it changes, not when it is first used

Parse and validate config at deploy or startup. A bad value read only when a rare path runs can sit unnoticed for days. Check for dangerous values automatically: empty allow/deny lists, wildcards, root paths, and changes far larger than usual.

### 2.3 Dormant paths are risky

Turning on a long-unused flag or code path is a risky change: test that path before enabling it.

### 2.4 Code and behavior roll out separately

Put risky behavior changes behind flags so they can be turned off without a redeploy, and so reverting config does not force reverting code. With the flag off, the binary behaves exactly as before.

---

## 3. Rollout

### 3.1 Stage it by failure domain

Deploy to one box or a small canary first (at most about 10 % of traffic) beside the current version, with metrics scoped to the canary; then one zone, cell, or region at a time — never all failure domains at once. Two zones in a region get a given deployment on different days.

- **Flag:** pipelines that deploy every region, zone, or cluster in parallel; a canary judged by fleet-wide metrics it cannot move.

### 3.2 Keep capacity while you deploy

Replace at most about a third of the instances at a time, so at least two thirds keep serving — services are sized to lose a zone anyway.

- **Flag:** `maxUnavailable` of 50 % or more on a fleet already running hot.

### 3.3 Gate each step, roll back automatically

After each step, check health signals — errors, latency, health checks, saturation, *and* the health of upstream and downstream services — and stop and roll back automatically on regression. Wait a bake time between steps (slow issues appear late or at peak) and require enough requests to have exercised the changed code before promoting.

- **Flag:** promotion as soon as pods are ready; no automatic rollback; gates that look only at the deployed service.

### 3.4 Report success only when the new code really works

Readiness should pass only once the process is listening, config and credentials are loaded, and it is serving correctly (for a consumer: joined its group and processing). A deploy script that reports success right after starting the process marches through the fleet breaking one server after another.

### 3.5 Emergency changes use the normal pipeline, faster

An emergency push is the regular push with its rate limits turned up, not a separate path. Separate hotfix paths are untested exactly when needed. Changes that reach many tenants or regions at once are rate-limited.

---

## 4. Health checks

### 4.1 Shallow checks for routing, deep checks elsewhere

Let the load balancer or orchestrator remove a host only for problems definitively local to it (process alive, disk, critical local resources). A deep check that calls a shared dependency (database, broker, object store) fails on every host at once when that dependency blips, and automation then removes or restarts the whole fleet.

- **Flag:** a readiness or liveness probe that calls the database, broker, object storage, or another service.

### 4.2 Fail open when everyone looks unhealthy

If deep checks do feed routing, the balancer must fail open when all hosts (or all in a zone) fail at once — that means the check is broken, not the fleet. Test partial dependency failures that make hosts flap.

### 4.3 A health check must not turn a soft dependency into a hard one

Failing the whole host because one dependency or one rarely used API is unhealthy takes down everything else it could still serve.

### 4.4 Health checks get answered under load

Reserve capacity: more worker threads than the proxy's maximum connections (a few extra up to double), or a concurrency limiter that always admits health checks.

### 4.5 Watch for black holes

A failing host often fails *fast* and so attracts more work (least-requests balancing, fast queue pollers). A consumer whose every message fails (full disk, expired credentials) should stop taking work, not poll at full speed.

---

## 5. Migrating to a new data store

Build confidence from three angles together: aggregate parity (at scale, do both stores return statistically similar answers?), explained differences (for each divergence, can you say *why*?), and adversarial probes (empty results, very large results, multi-key fan-out, stale data, races). Each alone hides what the others catch.

---

## Sources

- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/ensuring-rollback-safety-during-deployments/ — Ensuring rollback safety during deployments
- https://aws.amazon.com/builders-library/automating-safe-hands-off-deployments/ — Automating safe, hands-off deployments
- https://aws.amazon.com/builders-library/implementing-health-checks/ — Implementing health checks
- https://aws.amazon.com/builders-library/static-stability-using-availability-zones/ — Static stability using Availability Zones
- System Design for Interviews and Beyond (Mikhail Smarshchok), reliability module — confidence in a data store migration
