# Queries

Open this when writing or fixing panel queries. PromQL first; InfluxQL notes at the end.

## RED

Rate and errors on one panel, per component:

```promql
# A: success
sum by (container) (rate(uploads_total{namespace="$ns", outcome="success"}[$__rate_interval]))
# B: failure — 0 only for series that have successes
sum by (container) (rate(uploads_total{namespace="$ns", outcome="failure"}[$__rate_interval]))
  or (sum by (container) (rate(uploads_total{namespace="$ns", outcome="success"}[$__rate_interval]))) * 0
```

Never end the failure query with `or vector(0)`: when the metric does not exist it draws a red
"failure" line with no labels, and it hides which component the zero belongs to.

Success share for a table or stat (window larger than a scrape gap):

```promql
100 * sum(increase(uploads_total{outcome="success"}[1h]))
    / sum(increase(uploads_total[1h]))
```

## Duration

Percentiles and max on one panel (max dashed via an override on `.*max$`):

```promql
histogram_quantile(0.5,  sum by (le, container) (rate(x_duration_seconds_bucket{...}[$__rate_interval])))
histogram_quantile(0.95, sum by (le, container) (rate(x_duration_seconds_bucket{...}[$__rate_interval])))
histogram_quantile(0.99, sum by (le, container) (rate(x_duration_seconds_bucket{...}[$__rate_interval])))
histogram_quantile(1,    sum by (le, container) (rate(x_duration_seconds_bucket{...}[$__rate_interval])))
```

`histogram_quantile(1, …)` is the upper bound of the highest non-empty bucket, not the exact
max — say so in the description if it matters.

Heatmap of the same histogram:

```promql
sum by (le) (increase(x_duration_seconds_bucket{...}[$__rate_interval]))
```

with format `heatmap`, `calculate: false`, y-axis unit set, a single-hue scheme (Blues).

Rare calls (a dependency called a few times an hour): use a longer window
(`increase(...[1h])`) and say the number rests on few samples.

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

## Status 0 / 1 / 2

Worst of several checks. Each check is a `bool` comparison (×2 for the error level); give each
a distinct label so `or` keeps all of them, then take the max per component:

```promql
max by (container) (
     label_replace((<success %> < bool 99),           "check", "0", "", "")
  or label_replace((<success %> < bool 95) * 2,       "check", "1", "", "")
  or label_replace((<delay seconds> > bool 1800),     "check", "2", "", "")
  or label_replace((<delay seconds> > bool 3600) * 2, "check", "3", "", "")
)
```

Without the distinct label, `or` drops every right-hand series whose labels match one already
on the left, and the status silently ignores later checks.

Error share that treats "no failures" as 0 instead of "no data":

```promql
(sum(increase(calls_total{status!="OK"}[1h])) or vector(0)) / sum(increase(calls_total[1h]))
```

Here `or vector(0)` is correct: it is inside a ratio with no labels, not a plotted failure line.

## Consumer lag in time

Prefer lag in seconds from the lag exporter (for example Burrow's time-lag metric) over offsets;
people can compare seconds to an SLO. Link the consumer dashboard from the lag cell.

## Top-N

Per-partition metrics multiply quickly. Plot `topk(20, max by (topic, partition) (...))`
and say "top 20" in the title.

## InfluxQL notes

- `0 / 0` returns 0, not null. Write success as `100 * (1 - sum("failed") / (sum("ok") + sum("failed")))`
  so no traffic reads 100 %.
- Numerator / denominator pairs only give averages: no percentiles, no heatmaps.
- `lastNotNull` on sparse series can show a stale value; prefer `last` with `fill(none)` and say
  the window in the title.
