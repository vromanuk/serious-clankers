# Latency panels

Open this when building or fixing a duration panel or its heatmap.

## Question it answers

How long does the work take — typically, in the tail, and at worst — and has the shape
changed? Bad: p99 or max climbing toward the timeout, or a second band appearing in the
heatmap.

## Preferences

- For every histogram, two panels on the same line and the same height:
  - `"<Thing> duration: p50 / p95 / p99 / max"` — max dashed;
  - `"<Thing> duration heatmap"` — same buckets.
- Second in each component row, after rate and errors.
- If the source only has averages (numerator / denominator, or InfluxQL pairs), say
  percentiles and heatmaps are impossible and name the histogram it would need.

## Options

- Percentile panel: timeseries, unit `s`; override on `.*max$` with dashed line style.
- Heatmap panel: query format `heatmap`, `calculate: false` (buckets come from the
  histogram), y-axis unit set, single-hue color scheme (Blues).
- Grid: each `w: 12`, or `16 + 8` when the heatmap can be narrower.

## Recipe

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

Rare calls (a dependency called a few times an hour): use a longer window
(`increase(...[1h])`) and say the number rests on few samples.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Only "duration p95" | tail and worst case hidden | add p50, p99, max to the panel and a heatmap beside it |
| Max read as the exact slowest call | `histogram_quantile(1, …)` is a bucket bound | say "upper bucket bound" in the description |
| Slow calls flat at the top bucket | buckets stop below the timeout | recommend buckets past the timeout (`metrics.md`) |
| Heatmap asked for on an averages-only API | no buckets to draw | explain; name the histogram to export |
| Spiky percentiles for a rarely called dependency | `$__rate_interval` holds one or two calls | `increase(...[1h])`, note few samples |
| Heatmap re-buckets the data | `calculate: true` on pre-bucketed series | `calculate: false`, format `heatmap` |

## Check

- Every `_bucket` metric has both panels, on one line.
- `sum by` keeps `le`.
