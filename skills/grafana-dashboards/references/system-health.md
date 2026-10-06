# System health panel

Open this when building or fixing the "ready vs expected pods" panels in the open section.

## Question it answers

Is every component running the pods it should? Bad: ready below expected, restarts, crash
loops, or an unplanned scale change in the time range.

## Preferences

- Mirror the deploy view: per component, a timeseries of pods ready (blue) and expected
  (yellow `#FADE2A` line, no fill).
- Link the pod-detail dashboard from the panel and the header.
- No CPU or memory panels here or anywhere; the pod-detail dashboard has them.

## Options

- Timeseries; yellow series: fill opacity 0, fixed color.
- Grid: three panels of `w: 8, h: 6`, or one per component.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| CPU and memory timeseries per pod | habit | remove; link pod detail filtered to the component |
| Expected line hides the ready area | expected drawn with fill | no fill on the yellow line |
| Expected line green or blue | reference line given a status color | yellow, the reference-line color |
