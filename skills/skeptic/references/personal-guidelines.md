# Guidelines (review extract)

Short extract for stage agents. Depth: `write-code.md`, `testing.md`, `examples/`.  
Care priority when weighting findings lives in the skeptic coordinator skill.

## Optimize for the reader

Prefer simple to read over simple to write. Descriptive names, clear control flow, intent at the call site. Tedious for the author is fine if future readers move faster.

## Performance when it matters

Default: don’t sacrifice clarity for speculative speed. When perf is real (measured, SLO, DB/IO engine): a local, documented tradeoff is OK — comment the *why* / what not to undo.

## Architecture

- Component = job + namespace; not horizontal layers across the app.  
- Depend only on other components’ **public** surface.  
- One owner for data/tables.  
- Compose at the edge. Small tools stay local.  

## Thinking code vs shell

- Thinking: data in → decision out; no network/disk/env/clocks/random/threads inside.  
- Shell: read → decide → act.  
- Prefer sync pure cores; async at the edge.  
- **One function per task**, composed by a thin main — see `write-code.md` § Composition.  

## Tests

- New behavior → tests; assert outcomes, not mock call counts.  
- Prefer public-contract tests (stable under pure refactor).  
- Property/invariant tests when input or state space is large; snapshots when a stable blob is the contract.  
- Detail: `testing.md`.  

## Comments

- Aim for **local reasoning**: intent clear at the call site / API without reading every callee.  
- **Documentation** (`//!` / `///`): design and intent. **Implementation** (`//`): non-obvious choices and hazards — not line narration.  
- Only truths names/types/tests don’t make cheap. Why / who must / must not / negative space.  
- Non-trivial decisions: purpose + given/expected (**no** type line).  
- Prefer constraints over ticket IDs; tickets only when necessary (e.g. critical bug scar).  
- See `examples/comments.md` and `examples/design-headers.md`.  

## Language

- Plain words. No coined pattern nicknames in comments or names.  

## Delivery

- Prefer reviewable steps: land per chunk **or** one PR with **separate meaningful commits**.  
- Flag one opaque mega-commit / squash of many unrelated ideas.  

## Review output

- NUMBER issues, LETTER options; no “do nothing” option.  
- Pros / cons / gain / worse when. No time estimates.  
- Plain-text or component diagrams (no Mermaid required).  
