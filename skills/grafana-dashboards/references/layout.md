# Layout

Open this when sketching or reviewing the overall dashboard structure.

## The two questions

| Section | Question | Contents |
|---|---|---|
| Open | Is it working? Which part is broken? | canvas, system health, stats tables |
| Collapsed rows | Why? | one row per component: RED, then saturation and causes |

A reader who only looks at the open section should be able to say which component is unhappy.
A reader who opens that component's row should find the cause without leaving the dashboard,
or a link to the dashboard that has it.

## Plain-text sketch first

Agree the layout before building. Plain text only, no diagram languages.

```text
┌ Canvas (full width) ──────────────────────────────────────────────┐
│ Kafka → Relay A → Storage → API → Partners   (status on every box) │
└────────────────────────────────────────────────────────────────────┘
System Health:  [Relay A ready vs expected] [Relay B ...] [API ...]
Stats tables:   [Relays: health | rate | upload ok % | delay | lag | trend | logs]
                [API:    health | q/s  | success %   | latency          | logs]
▸ API            (collapsed)  rate+errors | latency | latency heatmap ...
▸ Relays         (collapsed)  rate+errors | duration | heatmap | backpressure ...
▸ Kafka Consumer (collapsed)  ...
```

## Grid

Grafana's grid is 24 columns wide; one row unit is about 30 px plus an 8 px gap.

- Canvas: full width (`w: 24`). Pick the height from the drawing's bottom edge plus the panel
  header; empty space under the drawing means the panel is too tall.
- System health: three panels of `w: 8, h: 6`, or one per component.
- Stats tables: `h` = header + one line per row + a little room; a two-row table needs about
  `h: 6`, a one-row table `h: 4`. A table that scrolls is squashed.
- Collapsed rows: at most three panels per line (`w: 8` each, or `12 + 12`, or `16 + 8`). A
  line of small stat panels (`w: 3` each) next to one graph is fine.
- Latency panel and its heatmap sit on the same line, same height.

## RED inside a component row

1. Rate and errors together: `"<Things>/s: success vs failure"`. Success blue, failure red.
2. Duration: `"<Thing> duration: p50 / p95 / p99 / max"` and `"<Thing> duration heatmap"`.
3. Saturation and causes: queue depth, backpressure share and reasons, in-flight vs limit,
   retries, dependency calls, storage capacity.

Prefer one line per question. If a component has no histogram, say so in the row rather than
leaving a gap.

## System health

Mirror the deploy view: per component, a timeseries of pods ready (blue) and expected
(yellow line, no fill). It shows restarts, crash loops, and scale changes over the time range.
Do not add CPU or memory here; link the pod-detail dashboard from the panel and the header.

## Stats tables

One row per component (or per instance of it). Columns, in this order:

| Column | Content | Cell |
|---|---|---|
| health | worst of the component's checks: 0 OK, 1 warning, 2 error | colored background, value mapping to "● OK" / "▲ WARN" / "✗ ERROR", link to the deploy tool for that app |
| name | plain-English component name | link to pod detail |
| rate | processing rate (records/s, req/s) | blue |
| errors | success % or failures/s | thresholds |
| duration / delay | latency or "store delay" | thresholds |
| lag | consumer lag in time | link to the consumer dashboard |
| trend | sparkline of the rate | |
| logs | "🔍 errors ↗" | link to the log search, filtered to the app and warning+ |

Build with instant table queries, one per column, then transformations: `timeSeriesTable`
(for the sparkline), `joinByField` on the component label, `organize` to rename, order, and
hide helper columns (keep them for links, hide with `custom.hidden`). Map label values to
display names with a value mapping rather than renaming in PromQL.

Text-link cells: a numeric helper column with a value mapping (`range` from -1e12 to 1e12 →
"🔍 errors ↗") plus a data link. Regex mappings do not apply to numeric cells.

## Variables

- One datasource variable the user picks (environment).
- Derive everything else hidden from it: environment name, short env, log index, matching
  secondary datasources (regex on datasource name), so switching environment switches every
  panel.
- A component filter (custom or label values) only if the collapsed rows need it.

Hidden variable derived from the chosen datasource's name (query variable, `hide: 2`):

```text
query:  query_result(label_replace(vector(1), "datasource", "${DataSource:text}", "", ""))
regex:  /datasource="prod-([a-z]+)-metrics"/          # captures "eng" or "prd"
```

A value that differs per environment (a log index, for example) nests a second
`label_replace` that sets it by matching the datasource name, then extracts it with a regex.

Secondary datasources follow by name: a datasource-type variable (`hide: 2`) with a regex such
as `/^tools-${env}-metrics$/` or `/^app-stats-${env}$/`.

## Header links

Pod detail per component, consumer dashboard, storage dashboard, runbook, and the API's own
dashboard. Name links the way people say them.

## Descriptions

Every panel gets one or two plain sentences: what it shows, what normal looks like, what bad
looks like. Example: "Share of time fetch was paused by backpressure while running. Shutdown is
not counted. Sustained above 20 % means the uploaders are the limit."

## Undeployed metrics

When a metric arrives with a later release, keep the panel and make it read "n/a" (see
queries.md). List those panels when reporting the dashboard, with the release that brings them.
