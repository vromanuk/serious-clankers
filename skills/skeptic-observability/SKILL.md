---
name: skeptic-observability
description: >
  Stage 5 of skeptic: production observability — structured logs, spans, metrics,
  panic hooks, async span attachment, alert rules. Use when skeptic runs or the
  user asks for telemetry/observability/alerting review only. Loads the
  observability skill for depth.
---

# Skeptic observability

**Question:** Can we see what this code does in production — useful logs/spans/metrics, without silent tracing bugs or dangerous metric labels — and do its alerts fire when someone must act, and only then?

## Load (on demand)

1. `../observability/SKILL.md` + `../observability/references/guide.md` — full rules  
2. `../observability/references/alerts.md` — when the diff adds or changes alert rules  

## Do

1. Apply the observability skill review checklist to the scoped diff.  
2. Focus on **new or changed** request paths, background jobs, error/panic paths, and metric sites.  
3. Check **units of work**: spans only for meaningful/variable chunks (not every helper); heavy I/O and real pipeline steps covered; duration + outcome where it matters (`guide.md` § Units of work).  
4. Flag: `println!` as prod log; unentered spans; undeclared `record` fields; spawn/async without instrument; unbounded metric labels; secrets in signals; panic only on stderr in services.  
5. **Alert rules** (new or changed): one line per alert on precision, recall, detection time, reset time, and how many alerts fire for one incident (`alerts.md` § The four properties). Then findings with path:line for: causes paged instead of symptoms; raw counts instead of ratios; averaged ratios; a long `for:` as the only noise filter; threshold steps that all page at once; low traffic with no guard; one alert per pod for one incident, or a summed ratio that hides one bad pod (suggest per-pod at ticket severity); signals that can vanish with no missing-data alert; missing runbook or context.  
6. New production paths or metrics with **no alert** where a failure would hurt users: note it as a question, not a demand.  
7. When telemetry is in scope: `observability: ok` (one line) or findings with path:line.  
8. Tiny pure refactors with no runtime surface → `none` with one-line why is fine.  

## Do not

- Pure unit-test design (→ testability / unit-tests stage)  
- Comment/naming style as the main pass (→ comments / naming)  
- Absolute ban walk except secrets overlap (→ hard-rules)  
- Invent a full observability platform redesign unless the diff already goes there  
- Alert routing, on-call policy, or SLO design beyond a short note  

## Output section title

`## 5. Observability`
