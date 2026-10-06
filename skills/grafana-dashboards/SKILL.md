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
4. **Open the panel file** for each panel you build or fix (table below).
5. **Generate the JSON from a script**, not by hand-editing exported JSON.
6. **Verify**: run every query, run the structure checks and each panel file's Check section,
   look at a screenshot. Report ok / empty / error counts and why each empty panel is empty.
7. **Ask before writing** to a shared folder or overwriting a dashboard you did not create.

Load `references/verify.md` for steps 5–6 (upload API, query check, screenshot, checks).
Load `references/local-setup.md` when working on Tesla energy telemetry dashboards.

## Panels

| Building or fixing | Open |
|---|---|
| Sections, collapsed rows, grid, variables, header links, descriptions | `references/structure.md` |
| Architecture canvas: boxes, arrows, bend points, text from queries | `references/canvas.md` |
| Stats table: columns, links, transformations, heights | `references/tables.md` |
| "X/s: success vs failure" timeseries | `references/rate-errors.md` |
| Percentiles + max panel and its heatmap | `references/latency.md` |
| Ready vs expected pods | `references/system-health.md` |
| OK / WARN / ERROR on a box, cell, or stat | `references/status.md` |
| A value that misleads on any panel | `references/honest-values.md` |
| Proposing or reviewing metric names, types, labels | `references/metrics.md` |

Each panel file has: the question it answers, preferences, Grafana options, a recipe, common
mistakes, and a check.

## Layout: RED first

Open section, top to bottom:

- **Architecture canvas** — the data flow with live status.
- **System health** — ready vs expected pods over time per component (expected as a yellow line).
- **Stats tables** — one row per component: health cell, rate, error rate or success %,
  latency, lag if it consumes a queue, a sparkline, and a logs link. Tall enough to show
  every row without scrolling.

Collapsed rows, one per component, in RED order:

1. **Rate + errors on one panel**: "Uploads/s: success vs failure" (success blue, failure red).
2. **Duration**: percentiles + max panel, heatmap beside it.
3. Then saturation and "why" panels (queues, backpressure, retries, dependency health).

At most **three panels per line** in a row (a line of small stat panels is fine). No CPU or
memory panels — link the pod-detail dashboard instead. Every panel gets a plain-words
description of what it shows and what "bad" looks like.

## Colors

- Blue `#5794F2` OK · orange `#FF9830` warning · red `#F2495C` error · yellow `#FADE2A`
  reference lines (expected, limit). **Never green.**
- Component identity colors (one per relay, dependency, store) must differ from status colors
  and never replace a visible status value.

## Duration

For every histogram: one panel with **p50, p95, p99, and max** (max dashed) and a **heatmap**
of the same buckets next to it. If the source only has averages, say percentiles and heatmaps
are impossible and name the histogram it would need.

## Canvas

Flow left to right. Every arrow lands on the **middle of an edge** and is **horizontal,
vertical, or a right-angle elbow** — never diagonal. Arrows dashed, animated, colored by a
status field, with room to see them. Boxes color-coded per component, full plain-English
names, status as a value inside. Show backpressure, what gets written where, and each external
dependency with status, call rate, and latency.

## Panel looks wrong

| Symptom | Open |
|---|---|
| Red failure line with no failures; 0 % success with no traffic | `rate-errors.md` |
| Always 0; "field not found"; alarming age; label names the wrong thing | `honest-values.md` |
| Squashed table; link cell shows a number | `tables.md` |
| Diagonal or skewed arrows; every box the same blue | `canvas.md` |
| Status ignores a failing check | `status.md` |
| Only p95; slow calls flat at the top bucket | `latency.md` |

The short rules: name what the counter counts; show a peak (`max_over_time`) for short-lived
activity; `or vector(-1)` + `-1 → n/a` for undeployed metrics; failure fallback
`or (<success expr>) * 0`, never `or vector(0)`; success ratios that read 100 % with no traffic.

## Metric design

When metrics are proposed or reviewed for a dashboard, check:

- **Names**: component prefix, base unit in the name, `_total` only on counters, one vocabulary
  for outcome values (`success`/`failure`).
- **Types**: counters for anything cumulative; a seconds counter for "share of time" instead of
  a 0/1 gauge; histograms whose buckets cover the timeout.
- **Labels**: bounded values only. No UUIDs, ids, DINs, timestamps, window starts, file names,
  or raw error text. Estimate series = product of label value counts × pods.

Report findings as recommendations. Do not edit application code unless asked.

## Never

- Never create or overwrite a dashboard in a shared folder, or touch another team's
  dashboard, without the user's go-ahead for that folder and uid.
- Never write session cookies, API tokens, or service-account keys into scripts, JSON,
  commits, or chat.
- Never query a metric, label, or value you have not confirmed exists.
- Never change application code to fix a metric finding unless asked.
- Never call a dashboard done without the query check, structure check, and a screenshot.
