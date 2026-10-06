# Stats tables

Open this when building or fixing a stats table in the open section.

## Question it answers

Which component is unhappy, and where do I click next? One row per component (or per
instance of it); a reader scans the health column, then follows a link.

## Preferences

Columns, in this order:

| Column | Content | Cell |
|---|---|---|
| health | worst of the component's checks (`status.md`) | colored background, value mapping to "● OK" / "▲ WARN" / "✗ ERROR", link to the deploy tool for that app |
| name | plain-English component name | link to pod detail |
| rate | processing rate (records/s, req/s) | blue |
| errors | success % or failures/s | thresholds |
| duration / delay | latency or "store delay" | thresholds |
| lag | consumer lag in time, if it consumes a queue | link to the consumer dashboard |
| trend | sparkline of the rate | |
| logs | "🔍 errors ↗" | link to the log search, filtered to the app and warning+ |

- Height shows every row without scrolling.
- Prefer lag in seconds from the lag exporter (for example Burrow's time-lag metric) over
  offsets; people can compare seconds to an SLO.

## Options

- Instant table queries, one per column.
- Transformations, in order: `timeSeriesTable` (for the sparkline), `joinByField` on the
  component label, `organize` to rename, order, and hide helper columns.
- Helper columns kept for links are hidden with `custom.hidden`, not removed.
- Display names come from a value mapping on the label, not from renaming in PromQL.
- Text-link cells: a numeric helper column with a value mapping (`range` from -1e12 to 1e12 →
  "🔍 errors ↗") plus a data link.
- Height: header + one line per row + a little room; a two-row table needs about `h: 6`, a
  one-row table `h: 4`.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Table scrolls ("squashed") | `h` too small for the rows | header + one line per row + room |
| Link text cell shows a number | regex mapping on a numeric cell (regex applies to strings only) | `range` mapping -1e12…1e12 → text |
| Link breaks after hiding a column | helper column removed instead of hidden | `custom.hidden` |
| Names differ between panels | renamed in PromQL per query | one value mapping on the label |
| InfluxQL cell shows a stale value | `lastNotNull` on a sparse series | `last` with `fill(none)`, window in the title |
| Lag hard to judge | offsets, not time | time lag; link the consumer dashboard |
| Undeployed column shows "field not found" | metric missing | `honest-values.md` § Not deployed yet |

## Check

- Every row visible in the screenshot without scrolling.
- Health, lag, and logs cells link where the table above says.
