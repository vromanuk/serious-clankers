# Build and verify

Open this when generating, uploading, or checking a dashboard.

## Dashboard as code

- A generator script (Python is fine) writes the dashboard JSON. Panels come from small
  builder functions (`timeseries`, `latency_panel`, `heatmap`, `rate_and_errors`,
  `stats_table`, canvas `element` / `connection` / `chip`) so a rule is fixed once.
- Keep thresholds, colors, window lengths, and component lists as named constants at the top.
- Keep the generator next to the code it describes, or in the dashboards repo; ask the user
  where before committing. Scratch output stays out of version control.

## Auth

Use whatever the user has: a service-account token (`Authorization: Bearer …`) or a browser
session cookie read from a local file. Read it at run time; never paste it into the script,
JSON, commits, or chat. Session cookies expire (often within half an hour): on HTTP 401, stop
and ask the user to refresh, then retry the same step.

Every HTTP call gets a timeout (`curl -m 60`).

## Upload

```text
POST /api/dashboards/db
{"dashboard": <json with fixed uid>, "folderUid": "<folder>", "overwrite": true,
 "message": "<ticket>: <what changed>"}
```

- Fixed `uid` so links survive uploads; `id: null` for a new dashboard.
- Upload only to a folder the user named, and only overwrite a uid you created.
- Report the returned `version` and URL.

## Query check

Run every panel query (and every query variable) against the target environment and count
results:

```text
POST /api/ds/query
{"from": "now-3h", "to": "now",
 "queries": [{"refId": "A", "datasource": {"uid": "<uid>", "type": "prometheus"},
              "expr": "<expr with variables filled>", "intervalMs": 60000, "maxDataPoints": 200}]}
```

- Substitute dashboard variables (`${DataSource}`, `$__rate_interval`, derived names) with real
  values for the environment before sending.
- Classify each query: **ok** (has values), **empty** (no frames or no values), **error**.
- Zero errors is required. Every empty query needs a reason: metric not deployed yet (name the
  release), no events in the window (no failures, no exceptions), or a wrong label — the last
  one is a bug to fix.
- Long runs can take a minute; run them in the background so they are not cut off.

## Structure check

On the generated JSON, before upload:

- No two panels overlap inside the same row, and no panel runs past column 24.
- Collapsed rows: at most three panels per line, except lines of small stat panels.
- Canvas: element names unique; every connection target exists; every target anchor is an edge
  middle; every arrow straight or with bend points; every arrow long enough to see. See
  canvas.md for the script.
- Every canvas field referenced by an element is produced by some query.
- Run the Check section of each panel file you used (canvas, tables, latency, rate and errors).

## Screenshot

The image renderer plugin is often not installed (render API returns 500). Use a headless
browser instead:

```python
# Playwright: log in with the same auth, open the dashboard in kiosk mode, screenshot.
context = await browser.new_context(viewport={"width": 1900, "height": 1400})
await context.add_cookies([{"name": "grafana_session", "value": cookie, "domain": host,
                            "path": "/", "secure": True, "httpOnly": True}])
page = await context.new_page()
await page.goto(f"{url}?orgId=1&from=now-3h&to=now&kiosk", wait_until="networkidle")
await page.wait_for_timeout(8000)          # canvas and tables finish drawing after network idle
await page.screenshot(path="top.png")
await page.mouse.wheel(0, 1300)
await page.screenshot(path="bottom.png")
```

Look at the image, not just the exit code: arrows, spacing, "n/a" chips, table heights, red
where nothing is wrong. Collapsed rows are not in the screenshot; expand them (click the row
title) when their layout changed.

If a screenshot looks unchanged after an upload, check the upload's returned version and take
a fresh screenshot before assuming the change failed.

## Reporting

When done, report:

- the link and version;
- query check counts and the reason for each empty panel;
- what stays empty until a release, and which release;
- anything not checked (for example "collapsed rows not screenshotted").
