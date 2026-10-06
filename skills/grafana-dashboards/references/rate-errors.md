# Rate and errors panel

Open this when building or fixing a component's "<Things>/s: success vs failure" timeseries.

## Question it answers

How much work is this component doing, and how much of it fails? Bad: the red line leaves
zero, or the blue line drops while traffic should be flowing.

## Preferences

- Success and failure on **one** panel, never two. Title: `"<Things>/s: success vs failure"`.
- Success blue `#5794F2`, failure red `#F2495C`. Never green.
- Name what the counter counts (see `honest-values.md`).
- First panel of every collapsed component row.

## Options

- Timeseries panel; unit `ops` or a custom unit naming the thing (`uploads/s`).
- Color per series with overrides matching the legend (`.*success.*` → blue,
  `.*failure.*` → red), fixed color mode.

## Recipe

```promql
# A: success
sum by (container) (rate(uploads_total{namespace="$ns", outcome="success"}[$__rate_interval]))
# B: failure — 0 only for series that have successes
sum by (container) (rate(uploads_total{namespace="$ns", outcome="failure"}[$__rate_interval]))
  or (sum by (container) (rate(uploads_total{namespace="$ns", outcome="success"}[$__rate_interval]))) * 0
```

Success share for a table or stat (window larger than a scrape gap):

```promql
100 * sum(increase(uploads_total{outcome="success"}[1h]))
    / sum(increase(uploads_total[1h]))
```

InfluxQL turns `0 / 0` into 0, not null. Write success as
`100 * (1 - sum("failed") / (sum("ok") + sum("failed")))` so no traffic reads 100 %.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| "Uploads/s" and "Upload failures/s" side by side | errors split from rate | one "Uploads/s: success vs failure" panel |
| Red failure line, no failures | `or vector(0)` on the failure query: draws an unlabelled line even when the metric does not exist | `or (<success expr>) * 0` |
| Failure zero cannot be tied to a component | `or vector(0)` drops the labels | same fix; the success expression keeps `container` |
| 0 % success with no traffic | InfluxQL 0 / 0 = 0 | `100 * (1 - fail / total)` |
| Success share jumps between 0 and 100 | `increase` window shorter than a scrape gap | widen the window (`[1h]`) |

## Check

- One query per outcome, both grouped by the same component label.
- No `or vector(0)` on a plotted failure series.
