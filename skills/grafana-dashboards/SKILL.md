---
name: grafana-dashboards
description: >
  Builds and reviews Grafana dashboards around RED (rate, errors, duration per component): an
  open overview (architecture canvas, system health, stats tables) that shows which part is
  broken, collapsed rows that show why, honest queries, and metric names/types/labels that can
  be graphed. Use when creating, redesigning, or reviewing a Grafana dashboard, panel, canvas,
  dashboard JSON or generator, when choosing metrics for a dashboard, or when a panel looks
  wrong (always 0, "field not found", fake failures, squashed tables, skewed arrows). Not for
  adding logs/spans in code alone (use observability) or alert routing.
spec_hash: f3d86b5b7c58
---

# Grafana Dashboards

A dashboard answers two questions in order: **is it working, and which part is broken?** (open,
at a glance), then **why?** (collapsed, one click down). RED — rate, errors, duration — is the
spine: every component gets those three before anything else.

## Workflow

1. **Name the components** in the data flow (sources, processors, dependencies, stores,
   consumers of the output). Each one gets RED.
2. **Confirm the metrics exist** for each component's rate, errors, and duration — query the
   datasource or read the code that emits them. Never write a metric, label, or label value
   you have not seen. Note which ones are not deployed yet.
3. **Sketch the layout in plain text** (open section, then one collapsed row per component) and
   agree it with the user before building.
4. **Generate the JSON from a script**, not by hand-editing exported JSON.
5. **Verify**: run every query, run the structure checks, look at a screenshot. Report
   ok / empty / error counts and why each empty panel is empty.
6. **Ask before writing** to a shared folder or overwriting a dashboard you did not create.

Load `references/verify.md` for steps 4–5 (upload API, query check, screenshot, checks).
Load `references/local-setup.md` when working on Tesla energy telemetry dashboards.

## Layout: RED first

Open section, top to bottom:

- **Architecture canvas** — the data flow with live status (see Canvas below).
- **System health** — ready vs expected pods over time per component (expected as a yellow line).
- **Stats tables** — one row per component: health cell (links to the deploy tool), rate,
  error rate or success %, latency, lag if it consumes a queue (links to the consumer
  dashboard), a sparkline, and a logs link for errors. Give tables enough height to show every
  row without scrolling.

Collapsed rows, one per component, in RED order:

1. **Rate + errors on one panel**: "Uploads/s: success vs failure" (success blue, failure red).
2. **Duration**: percentiles + max panel, heatmap beside it.
3. Then saturation and "why" panels (queues, backpressure, retries, dependency health).

At most **three panels per line** in a row (a line of small stat panels is fine). No CPU or
memory panels — link the pod-detail dashboard instead. Every panel gets a plain-words
description of what it shows and what "bad" looks like.

Load `references/structure.md` for grid sizes, variables, header links, and descriptions.

## Colors

- Blue `#5794F2` OK · orange `#FF9830` warning · red `#F2495C` error · yellow `#FADE2A`
  reference lines (expected, limit). **Never green.**
- Component identity colors (one per relay, dependency, store) must differ from status colors
  and never replace a visible status value.

## Duration

For every histogram: one panel with **p50, p95, p99, and max** (`histogram_quantile(1, …)`,
max dashed) and a **heatmap** of the same buckets next to it. If the source only has averages
(numerator/denominator), say percentiles and heatmaps are impossible and name the histogram it
would need.

## Canvas

- Flow left to right. Every arrow lands on the **middle of an edge** and is **horizontal,
  vertical, or a right-angle elbow** (connection bend points) — never diagonal.
- Arrows dashed, animated, colored by a status field.
- Boxes color-coded per component, full plain-English names ("Site Signals Relay", not
  "site"), status shown as a value inside. No wrapper lanes.
- Leave room: arrows must be clearly visible (roughly 50 px or more).
- Show backpressure, what gets written where ("one CSV + manifest per instance and window"),
  and each external dependency with status, call rate, and latency. Boxes link to their
  detail dashboards.

Load `references/canvas.md` for element and connection JSON, anchors, bend points, static
text, and a layout check script.

## Honest values

| Symptom | Cause | Fix |
|---|---|---|
| Label says "files" but counts uploads | name copied from intent | name what the counter counts ("chunks") |
| Gauge always 0 | activity lasts seconds, scraped rarely | `max_over_time(x[window])`, label "peak … (15 min)" |
| "Field not found" | metric not deployed yet | `… or vector(-1)` + mapping `-1 → n/a` |
| Red failure line, no failures | `or vector(0)` on the failure query | `or (<success expr>) * 0` |
| 0 % success with no traffic | InfluxQL 0 / 0 = 0 | `100 * (1 - fail / total)` |
| Age looks alarming at every close | age counts from window start | subtract the window length, show "delay" |

Load `references/honest-values.md` for these fixes; panel-specific queries live in the panel files.

## Metric design

When metrics are proposed or reviewed for a dashboard, check:

- **Names**: component prefix, base unit in the name, `_total` only on counters, one vocabulary
  for outcome values (`success`/`failure`).
- **Types**: counters for anything cumulative; a seconds counter for "share of time" instead of
  a 0/1 gauge; histograms whose buckets cover the timeout.
- **Labels**: bounded values only. No UUIDs, ids, DINs, timestamps, window starts, file names,
  or raw error text. Estimate series = product of label value counts × pods.

Report findings as recommendations. Do not edit application code unless asked.
Load `references/metrics.md` for naming rules, cardinality math, and worked examples.

## Never

- Never create or overwrite a dashboard in a shared folder, or touch another team's
  dashboard, without the user's go-ahead for that folder and uid.
- Never write session cookies, API tokens, or service-account keys into scripts, JSON,
  commits, or chat.
- Never query a metric, label, or value you have not confirmed exists.
- Never change application code to fix a metric finding unless asked.
- Never call a dashboard done without the query check, structure check, and a screenshot.
