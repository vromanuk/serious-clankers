# Metric design for dashboards

Open this when choosing, proposing, or reviewing metrics that a dashboard will graph. Report
findings as recommendations; changing application code is a separate request.

## Does the component have RED?

| Need | Metric | Type |
|---|---|---|
| Rate | `<component>_<things>_total{outcome}` | counter |
| Errors | same counter, `outcome="failure"` (not a second metric) | counter |
| Duration | `<component>_<operation>_duration_seconds` | histogram |

If one is missing, say which panel cannot be built and what metric would make it possible.

## Names

- **Prefix with the component or library**: `kafka_consumer_commit_duration_seconds`, not
  `commit_duration`. Unprefixed names such as `partition_consumer_lag` end up exported by
  dozens of apps; that is only fine if every one means exactly the same thing.
- **Base unit in the name**: `_seconds`, `_bytes`. Prometheus convention is seconds; mixing
  `_ms` and `_seconds` across a codebase makes every panel guess. With OpenTelemetry set the
  unit (`with_unit("s")`) and let the exporter add the suffix. A name with no unit
  (`publish_latency`) forces readers to open the code.
- **Counters end in `_total`** (OpenTelemetry's Prometheus exporter adds it). Do not put
  `_count` on a counter: it collides in meaning with a histogram's own `_count` series and
  becomes `x_count_total` after export.
- **One metric per meaning**: two counters for the same event (`rebalance_count` and
  `rebalances`) split the truth across panels.
- **Say what is counted**: a counter of window uploads is `chunks_put_total`, and the panel
  says "chunks", not "files".

## Types

- **Counter** for anything that only grows: requests, bytes read, records. A running total
  exported as a gauge breaks `rate()` semantics and reset handling.
- **Gauge** for a level that goes up and down: queue depth, partitions assigned, a timestamp
  ("loaded at", then `time() - x` in the panel).
- **Share of time** (paused, busy): a counter of seconds, `paused_seconds_total`, so
  `rate()` gives the fraction exactly. A 0/1 gauge sampled every 30 s misses short pauses.
- **Short-lived activity** (uploads in flight): a gauge reads 0 at most scrapes; prefer a
  busy-seconds counter, or graph the peak with `max_over_time`.
- **Histogram** for durations and sizes. Pick buckets from the real range: up to and past the
  timeout, dense where the SLO sits. Default buckets that stop at 10 s put a 25 s upload in the
  overflow bucket and hide it.
- **Ages**: export what people act on. "Delay past the window close" reads 0 when healthy;
  "age since window start" reads one window length at every close and needs correcting in
  every query.

## Labels and cardinality

Series count = product of the number of values of each label × number of pods (× histogram
buckets + 2 for histograms). Keep each label's value set small and known.

**Never as a label:**

| Value | Why | Put it in |
|---|---|---|
| UUIDs, device ids, DINs, customer ids, request ids | grows with the fleet; one series per id | logs, span fields, exemplars |
| timestamps, window starts, dates | new series every window, forever | the sample value or a log field |
| file names, object keys, URLs with ids | unbounded | logs |
| raw error messages | unbounded and noisy | an `error_kind` label with a fixed list, message in logs |
| pod names for long-lived aggregates | churns on every deploy | aggregate by container/deployment in queries |

**Fine as labels:** outcome (`success` / `failure`), operation or rpc name, reason from a fixed
enum, topic (a handful), status code class.

**Watch:** `topic × partition × consumer_group × pod` is bounded but multiplies fast; dashboards
should use `topk` and aggregate away `pod`.

**One vocabulary for outcomes** across a codebase: `success` / `failure` for results, a
separate `status` for protocol codes (gRPC `OK`, HTTP class). Mixing `ok` / `error`,
`success` / `failure`, and `kept` / `filtered` makes every query special.

## Estimating before shipping

```promql
count({__name__=~"kafka_consumer_.*"})                    # series for a prefix
count by (__name__) ({__name__=~"kafka_consumer_.*"})    # per metric
count(count by (partition) (partition_consumer_lag))      # values of one label
```

State the estimate in the review: "≈ 3 topics × 48 partitions × 2 pods = 288 series per gauge".

## Worked examples

These are patterns seen in real services. Use them to recognise the problem; they are not a
to-do list for any codebase.

| Seen | Problem | Recommendation |
|---|---|---|
| `chunks_put_duration_ms`, `rest_duration_ms` next to `grpc_request_latency_seconds` | mixed units | seconds everywhere for new metrics |
| `nats_publish_latency`, `consumer_batch_processor_request_latency` | no unit in the name | add the unit |
| `kafka_consumer_commit_count` beside `kafka_consumer_commit_duration_ms_count` | `_count` on a counter | `kafka_consumer_commits_total` |
| `kafka_consumer_rebalance_count` and `kafka_consumer_rebalances` | two metrics, one meaning | keep one |
| `partition_total_bytes_consumed` as a gauge | running total as gauge | counter |
| `windows_in_flight` gauge, uploads last seconds | always 0 when scraped | busy-seconds counter, or `max_over_time` in panels |
| `oldest_unstored_window_age_ms` from window start | panels must subtract the window | export delay past close |
| API latency as numerator/denominator | averages only | histogram |
| `outcome` values `kept`/`filtered`, `ok`/`error`, `success`/`failure` | mixed vocabulary | `success`/`failure` |
| `din_map_loaded_at_seconds` gauge | good: timestamp as value | keep |
| labels limited to outcome, status, topic, partition, rpc, reason | good: bounded | keep |
