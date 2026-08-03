---
name: skeptic-observability
description: >
  Stage 5 of skeptic: production observability — structured logs, spans, metrics,
  panic hooks, async span attachment. Use when skeptic runs or the user asks for
  telemetry/observability review only. Loads the observability skill for depth.
---

# Skeptic observability

**Question:** Can we see what this code does in production — useful logs/spans/metrics, without silent tracing bugs or dangerous metric labels?

## Load (on demand)

1. `../observability/SKILL.md` + `../observability/references/guide.md` — full rules  

## Do

1. Apply the observability skill review checklist to the scoped diff.  
2. Focus on **new or changed** request paths, background jobs, error/panic paths, and metric sites.  
3. Check **units of work**: spans only for meaningful/variable chunks (not every helper); heavy I/O and real pipeline steps covered; duration + outcome where it matters (`guide.md` § Units of work).  
4. Flag: `println!` as prod log; unentered spans; undeclared `record` fields; spawn/async without instrument; unbounded metric labels; secrets in signals; panic only on stderr in services.  
5. When telemetry is in scope: `observability: ok` (one line) or findings with path:line.  
6. Tiny pure refactors with no runtime surface → `none` with one-line why is fine.  

## Do not

- Pure unit-test design (→ testability / unit-tests stage)  
- Comment/naming style as the main pass (→ comments / naming)  
- Absolute ban walk except secrets overlap (→ hard-rules)  
- Invent a full observability platform redesign unless the diff already goes there  

## Output section title

`## 5. Observability`
