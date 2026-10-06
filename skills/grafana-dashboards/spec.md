# Grafana Dashboards

## Intent

Make the agent build Grafana dashboards that answer two questions in order: "is it working, and which part is broken?" at a glance, then "why?" one click down. RED — rate, errors, duration per component — is the spine of every dashboard: the overview shows RED per component, and each component's detail starts with its RED panels before anything else.

The skill also makes the agent check that the metrics behind a panel can tell the truth: honest labels, values that do not hide short-lived activity, fallbacks that do not invent failures, and metric names, types, and labels that keep series bounded. It complements `observability`, which covers emitting telemetry in code; this skill covers turning metrics into dashboards and the metric design dashboards depend on.

## Triggers

- **SHOULD** apply when creating, redesigning, or reviewing a Grafana dashboard, panel, canvas, or dashboard JSON / generator script.
- **SHOULD** apply when the user asks which metrics a dashboard needs, or reviews metric names, types, or labels so they can be graphed.
- **SHOULD** apply when the user reports a panel that looks wrong (always 0, "field not found", red line with no failures, squashed table, skewed arrows).
- **SHOULD NOT** apply to adding logs, spans, or panic hooks in code without a dashboard question (use `observability`).
- **SHOULD NOT** apply to alert-rule routing, on-call policy, or incident triage alone.
- **SHOULD NOT** apply to one-off ad hoc queries in Explore with no dashboard being built or changed.

## Behaviors

### Behavior: RED per component

The agent SHALL structure every dashboard around RED: for each component it SHALL show rate, errors, and duration, in that order, before saturation or internal "why" panels. Success and failure SHALL share one panel (rate and errors together), and duration SHALL come from a histogram when one exists.

#### Scenario: New dashboard for a relay and an API

- **GIVEN** a Kafka relay and an HTTP API that serves its output
- **WHEN** the user asks for a dashboard
- **THEN** the overview shows rate, error rate, and latency for both, and each component's detail row starts with a "requests/s: success vs failure" panel followed by its latency panels

#### Scenario: Errors on their own panel

- **GIVEN** a draft with "Uploads/s" and "Upload failures/s" as two separate panels
- **WHEN** reviewing the dashboard
- **THEN** the agent merges them into one "Uploads/s: success vs failure" panel with success blue and failure red

### Behavior: Overview first, why one click down

The agent SHALL put the "is it working, which part is broken" view open at the top — architecture canvas, system health (ready vs expected pods over time), and one stats table row per component with rate, error rate, latency, a health cell, and links — and SHALL put the "why" panels in collapsed rows, one row per component. Collapsed rows SHALL have at most three panels per line, except a line of small stat panels. The agent SHALL NOT add CPU or memory panels and SHALL link the existing pod-detail dashboard instead.

#### Scenario: Five panels on one line

- **GIVEN** a collapsed "Kafka Consumer" row whose last line holds five panels of widths 6, 6, 6, 3, 3
- **WHEN** reviewing the layout
- **THEN** the agent re-lays it out as lines of at most three panels

#### Scenario: CPU panel requested by habit

- **GIVEN** a draft with "CPU" and "Memory" timeseries for each pod
- **WHEN** reviewing the dashboard
- **THEN** the agent removes them and adds a header link and table link to the pod-detail dashboard filtered to that component

### Behavior: Status colors

The agent SHALL color status blue for OK, orange for warning, red for error, and yellow for reference lines (expected, limit, threshold), and SHALL NOT use green. Colors that identify a component (one per relay, dependency, or store) SHALL be distinct from the status colors and SHALL NOT replace a visible status value.

#### Scenario: Green OK

- **GIVEN** a stat panel whose thresholds go green → red
- **WHEN** reviewing colors
- **THEN** the agent changes the base step to blue, adds orange for warning, and keeps red for error

### Behavior: Latency with tail and shape

For every duration histogram, the agent SHALL show p50, p95, p99, and max (`histogram_quantile(1, …)`) on one panel, with max dashed, and SHALL place a heatmap of the same histogram next to it. When a source only reports averages, the agent SHALL say percentiles and heatmaps are impossible from it and name the missing histogram.

#### Scenario: Only p95 shown

- **GIVEN** a panel "Commit duration p95" built on `kafka_consumer_commit_duration_ms_bucket`
- **WHEN** reviewing the dashboard
- **THEN** the agent adds p50, p99, and max to that panel and a "Commit duration heatmap" panel beside it

#### Scenario: Averages-only API

- **GIVEN** an API whose metrics are a duration numerator and denominator
- **WHEN** the user asks for its latency heatmap
- **THEN** the agent explains only an average is possible and names the histogram the API would need to export

### Behavior: Architecture canvas

When the dashboard has a canvas, the agent SHALL draw the data flow left to right with every arrow landing on the middle of an edge and every arrow horizontal, vertical, or a right-angle elbow (using connection bend points) — never diagonal. Arrows SHALL be dashed, animated, and colored by a status field; boxes SHALL be color-coded per component with full plain-English names (no ambiguous short forms) and SHALL be spaced so every arrow is clearly visible. The canvas SHALL show backpressure, what gets written where (for example "one CSV + manifest per instance and window"), and each external dependency with status, call rate, and latency.

#### Scenario: Diagonal arrow to storage

- **GIVEN** two relay boxes at different heights both connected straight to one storage box
- **WHEN** reviewing the canvas
- **THEN** the agent puts storage level between them and gives each arrow two bend points so it runs horizontal, vertical, horizontal into storage's left middle

#### Scenario: Ambiguous chip labels

- **GIVEN** top-row chips labelled "site", "device", "site rec/s"
- **WHEN** reviewing the canvas
- **THEN** the agent renames them to the full component names, for example "Site Signals status" and "Site Signals rec/s"

### Behavior: Honest values

The agent SHALL make every panel say what its metric measures and SHALL make values survive real conditions:

- a label names what the counter counts (chunks, not files, when it counts chunks);
- a gauge of short-lived activity shows the peak over a window (`max_over_time`) or a derived delay, not a reading of now;
- a metric not deployed yet shows "n/a" (`or vector(-1)` plus a value mapping), not "field not found";
- a failure series falls back with `or (<success expr>) * 0`, never `or vector(0)`;
- a success ratio stays correct with no traffic (InfluxQL turns 0 / 0 into 0).

#### Scenario: Gauge always 0

- **GIVEN** an "uploaders in flight" chip reading `max(windows_in_flight)` while uploads take seconds every 15 minutes
- **WHEN** the user says it always shows 0
- **THEN** the agent switches it to `max(max_over_time(windows_in_flight[15m]))` and labels it "peak uploaders (15 min)"

#### Scenario: Red failure line with no data

- **GIVEN** a success-vs-failure panel whose failure query ends in `or vector(0)` and whose metric is not exported yet
- **WHEN** the user says it only shows failures
- **THEN** the agent replaces the fallback with `or (<success expr>) * 0` so the panel stays empty until the metric exists

### Behavior: Metric design that can be graphed

When reviewing or proposing metrics for a dashboard, the agent SHALL check names (component prefix, base unit in the name, `_total` only on counters, one vocabulary for outcome values), types (counters for anything cumulative, a seconds counter for "share of time" instead of a 0/1 gauge, histograms with buckets that cover the timeout), and labels (bounded values only — no UUIDs, ids, timestamps, window starts, file names, or raw error text) and SHALL estimate the series count. Findings SHALL be reported as recommendations; the agent SHALL NOT change application code unless asked.

#### Scenario: UUID label

- **GIVEN** a proposed counter `rows_written_total{instance_id=<uuid>}`
- **WHEN** reviewing metric design
- **THEN** the agent flags unbounded cardinality, estimates series as instances × pods, and suggests dropping the label and logging the id instead

#### Scenario: Cumulative value as a gauge

- **GIVEN** `partition_total_bytes_consumed` declared as a gauge holding a running total
- **WHEN** reviewing metric design
- **THEN** the agent recommends a counter so `rate()` and resets behave, without editing the code

### Behavior: Verified dashboard as code

The agent SHALL build dashboards from a generator script that writes the JSON, SHALL run every panel query against the target environment and report ok / empty / error counts with a reason for each empty query, SHALL check structure (no overlapping panels, unique canvas element names, every connection target exists, every arrow straight or elbowed), and SHALL look at a rendered screenshot before saying the dashboard is done.

#### Scenario: Ready to upload

- **GIVEN** a regenerated dashboard JSON
- **WHEN** the agent is about to report the dashboard as finished
- **THEN** it has run the query check, the structure check, and a screenshot, and reports which panels stay empty and why (for example "metric arrives with the next image")

## Constraints

### Constraint: No overwrite without consent

The agent MUST NOT create or overwrite a dashboard in a shared folder, or change another team's dashboard, without the user's go-ahead for that folder and uid.

### Constraint: No secrets in artifacts

The agent MUST NOT write Grafana session cookies, API tokens, or service-account keys into generator scripts, dashboard JSON, commits, or chat output.

### Constraint: No invented metrics

The agent MUST NOT put a metric name, label, or label value in a query without first confirming it exists in the target datasource or in the code that emits it.

### Constraint: No code changes from a dashboard review

The agent MUST NOT edit application code to fix metric design findings unless the user asks for that change.

<!-- skillet-version: 1.8.0 -->
