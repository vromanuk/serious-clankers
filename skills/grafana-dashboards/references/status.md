# Status values

Open this when a canvas box, a table health cell, or a stat shows OK / WARN / ERROR.

## Preferences

- Status is a number per component: 0 OK, 1 warning, 2 error.
- Value mapping: `0 → "● OK"`, `1 → "▲ WARN"`, `2 → "✗ ERROR"`.
- Thresholds: blue `#5794F2` base, orange `#FF9830` at 1, red `#F2495C` at 2. Never green.
- Status is a value shown inside the box or cell, not a border or a component identity color.

## Recipe

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

Error share that treats "no failures" as 0 instead of "no data":

```promql
(sum(increase(calls_total{status!="OK"}[1h])) or vector(0)) / sum(increase(calls_total[1h]))
```

Here `or vector(0)` is correct: it is inside a ratio with no labels, not a plotted failure line.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Status ignores a failing check | `or` drops right-hand series whose labels match the left | a distinct `check` label per comparison |
| OK shown green | default thresholds | blue base step |
| Every healthy box the same blue border | status put on the border | status as a value inside; border is component identity |
| Status "no data" when nothing failed | failure count has no series | `or vector(0)` inside the ratio, as above |
