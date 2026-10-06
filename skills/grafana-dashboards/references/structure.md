# Dashboard structure

Open this when sketching or reviewing the overall dashboard: sections, rows, grid, variables,
header links, and descriptions. Each panel's own rules live in its panel file.

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

- Collapsed rows: at most three panels per line (`w: 8` each, or `12 + 12`, or `16 + 8`). A
  line of small stat panels (`w: 3` each) next to one graph is fine.
- Panel sizes: canvas in `canvas.md`, system health in `system-health.md`, stats tables in
  `tables.md`, latency and heatmap in `latency.md`.

## RED inside a component row

1. Rate and errors together (`rate-errors.md`).
2. Duration: percentiles + max, and the heatmap beside it (`latency.md`).
3. Saturation and causes: queue depth, backpressure share and reasons, in-flight vs limit,
   retries, dependency calls, storage capacity.

Prefer one line per question. If a component has no histogram, say so in the row rather than
leaving a gap.

No CPU or memory panels anywhere; link the pod-detail dashboard instead.

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
