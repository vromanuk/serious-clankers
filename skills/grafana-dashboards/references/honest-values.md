# Honest values

Open this when a value on any panel misleads: always 0, "field not found", a scary age, a
label that names the wrong thing, or too many series to read. Panel-specific traps live in the
panel files.

## Name what is counted

A label says what the counter counts. A counter of window uploads is "chunks", not "files",
even if the intent was files.

## Short-lived activity

A gauge that is non-zero for seconds (uploads in flight, queue that drains) reads 0 at almost
every scrape. Show the peak over one cycle:

```promql
max(max_over_time(windows_in_flight{...}[15m]))
```

Label it "peak … (15 min)". Better, if you own the code: a counter of busy seconds, then
`rate(busy_seconds_total[5m]) / workers` is exact utilization.

## Ages and delays

An age measured from a window's start reads one window length at every close, even when all is
well. Show the delay past the close instead:

```promql
clamp_min(max(max_over_time(oldest_unstored_window_age_ms{...}[15m])) - 900000, 0)
```

Keep the window length as a named constant next to the query, with a note that it must match
the app's setting.

Time since an event exported as a timestamp gauge (good pattern):

```promql
time() - max(din_map_loaded_at_seconds{...})
```

## Not deployed yet

```promql
max(new_metric{...}) or vector(-1)
```

plus a value mapping `-1 → "n/a"` (muted color). For a table column per component, keep the
labels: `<expr> or (max by (container) (some_existing_metric{...}) * 0 - 1)`.

Keep the panel; list it when reporting the dashboard, with the release that brings the metric.

## Too many series

Per-partition metrics multiply quickly. Plot `topk(20, max by (topic, partition) (...))`
and say "top 20" in the title.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Label says "files" but counts uploads | name copied from intent | name what the counter counts ("chunks") |
| Gauge always 0 | activity lasts seconds, scraped rarely | `max_over_time(x[window])`, label "peak … (15 min)" |
| "Field not found" | metric not deployed yet | `… or vector(-1)` + mapping `-1 → n/a` |
| Age looks alarming at every close | age counts from window start | subtract the window length, show "delay" |
| Unreadable spaghetti of partitions | every series plotted | `topk(20, …)`, "top 20" in the title |

Failure-line and success-ratio traps: `rate-errors.md`.
