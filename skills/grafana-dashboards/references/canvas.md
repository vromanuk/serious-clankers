# Canvas

Open this when drawing or fixing an architecture canvas panel. Checked against Grafana 11–13.

## Question it answers

One picture of the data flow with live status: which box is unhappy, where data backs up, and
what leaves the system. It is the first thing on the dashboard, so it must read in seconds.

## Preferences

**Flow**

- Left to right: sources → processors → stores → consumers of the output. Put a shared
  dependency (an API both processors call) between the boxes that use it.
- No wrapper lanes around groups of boxes.

**Arrows**

- Land on the **middle of an edge** of the target.
- Run horizontal, vertical, or as a right-angle elbow. Align box centers so most arrows are
  naturally straight; use bend points for the rest. Never diagonal.
- Dashed and animated, colored by a status field (lag, failures, paused share).
- About 50 px or more of visible arrow between boxes.

**Boxes**

- Color-coded by component (fill + matching border + title color). Status is a value inside
  the box (`status.md`), not the border color — otherwise every healthy box is the same blue.
- Full plain-English names everywhere, including top-row chips ("Site Signals rec/s", not
  "site rec/s").
- Tall enough for title, subtitle, and a row of chips.
- Each box links to its detail dashboard (pod detail, consumer, storage).
- Small logos (a white tile with the product's SVG) help recognition; keep them few.

**Content**

- Backpressure (a box under the fetch step with share paused and top reason).
- What is written where ("one CSV + manifest per instance and window").
- Each external dependency with status, call rate, and latency.
- Storage capacity, and the output's consumers.
- A one-paragraph note under the drawing says what the main component does, in plain words.

## Options

### Coordinates

Element placement is in pixels from the top-left: `left`, `top`, `width`, `height`. Canvas
does not stretch the drawing to the panel, so draw on a fixed grid and scale x by
(panel width ÷ grid width) when generating if the drawing should fill the panel.

Panel size: full width (`w: 24`); height from the drawing's bottom edge plus the panel header.

Connection anchors are relative to the element: x from -1 (left) to 1 (right), y from 1 (top)
to -1 (bottom).

```text
LEFT   = (-1, 0)    RIGHT  = (1, 0)
TOP    = (0, 1)     BOTTOM = (0, -1)
```

An anchor off the middle is fine for the **start** of an arrow when it keeps the arrow level,
for example leaving a tall box's left edge at the height of a smaller box beside it:
`(-1, -drop / (height / 2))`.

### Elements

```json
{
  "type": "rectangle",
  "name": "site-relay",
  "config": {"align": "left", "valign": "middle", "size": 15,
             "color": {"fixed": "#B877D9"},
             "text": {"mode": "fixed", "fixed": "Site Signals Relay"}},
  "background": {"color": {"fixed": "#2e2546"}},
  "border": {"color": {"fixed": "#B877D9"}, "width": 2},
  "constraint": {"horizontal": "left", "vertical": "top"},
  "placement": {"left": 686, "top": 105, "width": 432, "height": 200, "rotation": 0},
  "links": [{"title": "Pod Detail", "url": "...", "targetBlank": true}],
  "oneClickMode": "link",
  "connections": []
}
```

- `metric-value` shows a field: `"text": {"mode": "field", "field": "site_rec"}` and
  `"color": {"field": "site_rec"}` so thresholds color it.
- `cloud` works for object storage; `rectangle` with `background.image` (`size: contain`) for
  logos.
- Element names must be unique; connections refer to them by name.

### Connections

```json
{
  "targetName": "storage",
  "source": {"x": 1, "y": 0},
  "target": {"x": -1, "y": 0},
  "vertices": [{"x": 0.5, "y": 0}, {"x": 0.5, "y": 1}],
  "color": {"field": "site_ok"},
  "direction": {"mode": "fixed", "fixed": "forward"},
  "lineStyle": {"style": "dashed", "animate": true},
  "path": "straight",
  "size": {"fixed": 2, "min": 1, "max": 6}
}
```

Bend points (`vertices`) are fractions of the way from the arrow's start point to its end
point: `x` of the horizontal distance, `y` of the vertical. Common shapes:

| Shape | Vertices |
|---|---|
| horizontal → vertical (into a TOP or BOTTOM anchor) | `(1, 0)` |
| vertical → horizontal (into a LEFT or RIGHT anchor) | `(0, 1)` |
| horizontal → vertical → horizontal (two boxes at different heights into one box's side) | `(0.5, 0)`, `(0.5, 1)` |

### Text from queries

- Static text that should still come from data (so it can include variables): an instant table
  query `label_replace(vector(1), "storage_text", "Object Storage\\ns3://bucket-${env}", "", "")`,
  then `"text": {"mode": "field", "field": "storage_text"}`.
- A top reason as text: `label_replace(topk(1, sum by (reason) (increase(x_total[15m]))),
  "relay_reason", "$1", "reason", "(.*)") or label_replace(vector(0), "relay_reason", "no pauses", "", "")`.
- Field names must be unique across all queries; name each query's legend after its field.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Diagonal arrow into storage | two boxes at different heights wired straight to one box | put storage level between them; bend points `(0.5, 0)`, `(0.5, 1)` into its left middle |
| Arrow lands on a corner or off-center | target anchor not an edge middle | target `x == 0` or `y == 0` |
| Arrows barely visible | boxes packed too close | ≥ 50 px of arrow; move boxes or shrink them |
| Every healthy box the same blue | status drawn as border color | border = component color; status as a value inside |
| Chips read "site", "device" | short forms | full names ("Site Signals status") |
| Empty band under the drawing | panel taller than the drawing | height = drawing bottom + header |
| Drawing hugs the left of a wide panel | canvas does not stretch | scale x by panel width ÷ grid width |
| Element shows the wrong value | two queries produce the same field name | unique field names, legend = field |
| Box missing its static subtitle per environment | fixed text cannot hold variables | text from a `label_replace(vector(1), …)` query |

## Check

Before uploading, compute each arrow's start and end in pixels and assert it is straight or
has bend points:

```python
def point(element, anchor):
    p = element["placement"]
    return (p["left"] + (anchor["x"] + 1) / 2 * p["width"],
            p["top"] + (1 - anchor["y"]) / 2 * p["height"])

for element in elements:
    for connection in element["connections"]:
        target = by_name[connection["targetName"]]          # also proves the target exists
        start, end = point(element, connection["source"]), point(target, connection["target"])
        is_straight = abs(start[0] - end[0]) < 2 or abs(start[1] - end[1]) < 2
        assert is_straight or connection.get("vertices"), (element["name"], target["name"])
        assert abs(start[0] - end[0]) + abs(start[1] - end[1]) >= 40, "arrow too short to see"
```

Also assert element names are unique, target anchors have x == 0 or y == 0 (edge middle), and
every field an element references is produced by some query.
