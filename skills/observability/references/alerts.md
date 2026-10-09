# Alert rules

Rules for **writing and reviewing** alert rules (Prometheus-style PromQL; the ideas apply to any alerting system).

**Sources:** Google SRE Workbook, ch. 5 "Alerting on SLOs" (Thurgood et al.); Prometheus docs, "Alerting rules" and alerting practices. Points marked *(general practice)* come from neither source; keep them, but do not cite them as the book's.

---

## What an alert is for

An alert asks a human to act on a **significant event**: something that hurts users now, or will soon if nobody acts. Pages and tickets are the only two ways to get a human to act. If nobody needs to act, it belongs on a dashboard, not in an alert.

Fewer alerts are better. Every alert that fires without needing action teaches on-call to ignore the next one.

---

## The four properties (the review questions)

| Property | Question | 100% means |
|----------|----------|------------|
| **Precision** | Of the alerts that fire, how many were significant events? | Every alert was worth acting on |
| **Recall** | Of the significant events, how many fired an alert? | Nothing significant was missed |
| **Detection time** | How long from the start of the problem to the notification? | — (shorter is better; should shrink as the problem gets worse) |
| **Reset time** | How long does the alert keep firing after the problem is fixed? | — (shorter is better; long reset confuses people or gets ignored) |

Every design choice below trades these against each other. A review states the trade for each new or changed alert, plus how many alerts fire for one incident:

```text
<alert>: precision <ok|weak> (<why>) · recall <ok|weak> (<why>) · detection <time, and does it scale with severity> · reset <time> · fires: <1 per service | 1 per pod, up to N | unbounded>
```

---

## Without an SLO

Most services have no SLO yet. The checks still apply:

- A threshold implies a target. **Write the target down** in the alert description ("we accept up to 1% failed requests"). It can become an SLO later without guessing.
- Base thresholds on **history or real pain** (what users noticed, what broke last time), not round numbers.
- With an SLO, the book's thresholds are **burn rates** (how fast the error budget is spent): page at 14.4× over 1h (2% of a 30-day budget) or 6× over 6h (5%); ticket at 1× over 3 days (10%). Without one, the same shape still works: a high ratio over short windows pages, and a low ratio over long windows opens a ticket.

---

## Checks

### 1. Symptoms, not causes

Page on what users or downstream consumers feel:

| Kind of system | Page on |
|----------------|---------|
| Online serving (request/response) | Error ratio and latency, at the top of the stack |
| Pipeline / stream consumer | How long data takes to get through (lag in **time**, freshness of output) |
| Batch / scheduled job | Time since the last **success**, compared with a few run intervals |
| Capacity (disk, queue limit, quota) | Time until the limit is hit, early enough to act |

Cause metrics (CPU, memory, restarts, queue depth on its own) belong on dashboards or in tickets, unless they directly predict user harm soon (for example, a stream will hit its limit and start rejecting writes).

**Page once per stack.** One layer pages for a user-visible failure, not every hop in the dependency chain.

### 2. Ratios, not raw counts

- `rate(errors[5m]) > 0.2` means something different at peak and at night. Prefer **bad / total**.
- Compute the ratio as `sum(rate(bad[w])) / sum(rate(total[w]))`. Never `avg()` of per-instance ratios: an idle instance with 1 failed request out of 1 weighs as much as a busy one. Apply `rate()` before `sum()`.
- Prefer lag in **time** (seconds behind) over lag in messages: a message count means different things at different throughputs.

### 3. Window, `for:`, and `keep_firing_for:`

| Choice | Detection | Precision | Reset | Note |
|--------|-----------|-----------|-------|------|
| Short window alone (e.g. 5m) | Fast | Low | Fast | Fires on blips; the book's example could alert 144 times a day and still meet the SLO |
| Long window alone (e.g. 6h) | OK | Good | Slow | Keeps firing for most of the window after recovery; long ranges are expensive to evaluate |
| Long `for:` as the noise filter | Slow, same for a 100% outage as a 0.2% one | Better | — | Timer resets when the value dips once: errors that come and go may **never** alert (poor recall) |
| **Long AND short window** | Good | Good | Good | Recommended; short window ≈ 1/12 of the long one (1h + 5m, 6h + 30m) |

- **Keep `for:` short**: long enough to absorb one bad scrape or evaluation (a minute or two), not the main noise filter. Get precision from the window length.
- **No `for:`** means the rule fires on its first true evaluation. That is fine when the expression is already sustained (time since last success, a long window). It is noisy on an instant rate over a short window.
- **`keep_firing_for:`** stops flapping and false resolutions during short data gaps, but adds directly to reset time. Keep it short and say why it is there.
- *(general practice)* Make a `rate()` range at least 4× the scrape interval, so one missed scrape does not empty the window.

Two windows, without an SLO:

```promql
# Fires when more than 5% of checkout requests fail, over the last hour and right now.
# Target: we accept up to 5% failed requests.
(
  # Long window: the failures have lasted long enough to matter, not a one-scrape blip.
  sum(rate(requests_failed_total{service="checkout"}[1h])) / sum(rate(requests_total{service="checkout"}[1h])) > 0.05
and
  # Short window: the failures are still happening, so the alert resolves minutes after a fix.
  sum(rate(requests_failed_total{service="checkout"}[5m])) / sum(rate(requests_total{service="checkout"}[5m])) > 0.05
)
```

Why both: the 1h window alone keeps firing for most of an hour after the fix; the 5m window alone fires on every blip. Requiring both means the problem has lasted **and** is still going. A severe outage crosses both thresholds within minutes, so detection still scales with severity.

### 4. Severity follows how fast someone must act

- Severe and fast → **page**. Low and sustained → **ticket**.
- A ladder of thresholds on the same window (5% / 10% / 25% / 50%) is not severity by speed: during a full outage every step fires at once. Prefer one paging rule (high ratio, short windows) and one ticket rule (low ratio, long windows), or suppress the lower steps while a higher one fires (Alertmanager inhibition).

### 5. Low traffic

At 10 requests an hour, one failure is a 10% error rate. Options from the book:

- **Synthetic traffic** (probes, black-box checks), so there is always a signal. Downside: if real users fail and synthetic requests succeed, the synthetic successes hide the failure.
- **Combine related services** that share a failure domain into one alert. Downside: one small service failing completely may not move the group; keep a long-window alert per service for that.
- **Make one failure matter less**: client retries with backoff, fallback paths.
- **A lower target or a longer window**, if one failed request really does not need a human.

*(general practice)* A **minimum-volume condition** (`and sum(rate(requests_total[w])) > N`) stops one failure from paging. It also hides a total outage that drops traffic to zero, so pair it with a check for missing traffic (check 7).

A tail latency quantile over few requests is noisy: p99 over 5 minutes with 10 requests is about the slowest single request.

### 6. How many alerts fire, and at what level

Every series the expression returns is **one alert**. The labels left after aggregation decide the count:

| Expression shape | Alerts when the whole service fails (10 pods) |
|------------------|-----------------------------------------------|
| `sum(rate(bad[w])) / sum(rate(total[w]))` | 1 for the service |
| `sum by (pod) (...) / sum by (pod) (...)` | 10, one per pod |
| No aggregation (pod, route, status, … all kept) | pods × routes × …, a flood |

Ask: **when this is true, how many alerts fire, and is that the number we want?**

**Service-level** (summed across pods): matches what users feel and fires once. It can miss one bad pod: with 20 pods, one pod at 100% errors is about 5% overall and may stay under the threshold. A bad node, a stuck connection pool, or a broken local cache is a real failure this alert can miss.

**Per-pod**: catches that pod, at a cost:
- A full outage fires N alerts for one incident.
- Each pod sees a slice of the traffic, so the low-traffic problem (check 5) gets worse.
- A restart gives the pod a new name, so it becomes a new alert: the `for:` timer starts over, and the old pod's alert lingers until its series goes stale.

**When to suggest a per-pod alert:** when the error ratio is summed across pods and pods can fail on their own (local state, connection pools, node problems). Usually at ticket severity, written as one of:
- The pod's own ratio is over a threshold, with a minimum request count per pod.
- The pod's ratio is well above the service ratio (e.g. 5× higher). This catches one pod that stands out and stays quiet during a full outage.

If both exist, suppress the per-pod alerts while the service-level one fires (inhibition, or an `unless` on the service-level condition), so a full outage gives one page, not N+1.

**Skip per-pod** when pods share every failure mode (stateless, same downstream) and health probes already restart a bad pod, or when existing pod-level alerts (crash loops, restarts) cover it.

**Watch for disguises:**
- `max by (service) (per-pod ratio)` fires once but on the **worst pod**: it is a per-pod alert named like a service alert. Say which one it is in the name and description.
- `count(... > threshold) > 0` collapses many series into one alert and drops their labels. Then the description must say how to find which ones.

### 7. Missing data

When a series disappears (process down, scrape failing, metric renamed), comparisons return nothing, and the alert never fires or quietly resolves. A ratio with zero traffic has no value at all. This is a recall gap.

- For signals that should always exist, add `absent()` / `absent_over_time()`, or a "no traffic" alert, next to the threshold alert.
- Make sure the scrape itself is watched (`up == 0`), and that something watches the alerting pipeline (Prometheus and Alertmanager themselves).

### 8. Every alert is actionable and carries its context

- `summary` with the current `$value` and the labels that say **where** (`$labels.pod`, table, topic, …).
- `description`: what it means for users and the target it protects.
- Runbook link with a first step. If the runbook's answer is "wait" or "ignore", delete the alert or demote it to a ticket.
- Dashboard link that lands on the failing part.
- **Comments in the expression.** One line above it: when the alert fires, in plain words, and the target it protects. When the expression has more than one part (two windows, a volume guard, `unless`, `and on()`), one comment per part saying what that part guards against. On-call should know why it fired without decoding the PromQL. PromQL accepts `#` comments, including inside a YAML `expr: |` block.

### 9. Few rules, same parameters, simple expressions

- Pick a small set of alert classes (window + threshold + severity) and apply them across services. The book advises against tuning windows and thresholds per service: it does not scale.
- Repeated sub-expressions (the same ratio in several alerts) → a recording rule, so every alert uses the same definition.
- The same rule copied per environment or per instance drifts. When one copy changes, check every copy.

### 10. Validate before deploying

- `promtool check rules` for syntax; `promtool test rules` for behavior: it fires on a sustained problem, stays quiet on a one-scrape blip, and stops soon after recovery.
- If rules are generated from templates, render them and check the rendered output.
- Keep the default evaluation interval; go faster only when detection time needs it.

---

## Not always a smell

- No `for:` on an expression that is already sustained (time since last success, long windows).
- A raw count alert when any single event is significant and rare (data corruption detected, a security event).
- A cause alert that predicts user harm before it happens (capacity running out), as long as it says how long is left.
- A per-pod alert without a service-level one, when the service is a single replica.

---

## Review flags

| Smell | Prefer |
|-------|--------|
| Pages on CPU / memory / restarts with no user impact | Symptom alert; cause on a dashboard or ticket |
| Raw error count threshold | Ratio of bad to total |
| `avg()` of per-instance ratios | `sum(bad) / sum(total)` |
| Long `for:` as the only noise filter | Longer window, or long + short window |
| Short window alone on a ratio, paging | Add a long window, or demote to ticket |
| Stepped thresholds on one window, all paging | One page rule (short windows) + one ticket rule (long windows), or inhibition |
| No minimum volume on a low-traffic ratio | Volume condition + missing-traffic check, or synthetic traffic |
| p99 on very few requests | Longer window, lower quantile, or volume condition |
| Unaggregated expression (one alert per series) | Aggregate to the level someone acts on |
| Service-level sum where pods fail independently | Suggest a per-pod or pod-vs-service alert at ticket severity |
| Per-pod and service alerts both page during an outage | Inhibit per-pod while the service alert fires |
| `max by` / `count` hides which instance | Name it honestly; put the "where" in the description |
| Signal that should always exist, no `absent()` / no-traffic alert | Add one |
| No runbook, or a runbook that says "ignore" | Add a first step, or delete / demote |
| Expression with several parts (windows, volume guard, `unless`) and no comments | One line above saying when it fires and the target; one comment per part saying what it guards against |
| Rule copied per environment with diverging thresholds | One definition, or check every copy |
