---
name: design-components
description: >
  Design or grow a job-shaped component: one public struct whose methods
  orchestrate use cases once; private interior uses pure-core/shell (sans-IO).
  Not app-wide folder layers and not a bag of stray public helpers or free
  orchestration functions. Use when creating a module/crate for a job, fixing
  shallow helper dumps, or when the user asks to design a component. Not for
  full multi-stage code review alone (use skeptic).
---

# Design components

**One story:** component-based architecture + pure-core/shell.

| Piece | Meaning |
|-------|---------|
| **Component** | One **job**, one namespace |
| **Public face** | Prefer **one struct** for the job; **methods** are use cases (`allocate`, `issue_invoice`). Hold deps on the struct. |
| **Private interior** | **Thinking** (pure rules) + **shell** (IO) + helpers — mostly unexported |

Cosmic Python’s “service layer” ≈ **that public struct**. We do **not** default to their full DDD stack (repo/UoW/aggregate ceremony).

## Load

1. `references/guide.md` — short merge (required for non-trivial design)  
2. `../skeptic-architecture/references/components.md` — layout rules  
3. `../skeptic-testability/references/pure-core.md` — thinking vs shell samples  

## Do

1. **Name the job** — not a technical layer.  
2. **Public struct** for the job; **list methods** = use cases (default exports).  
3. **Deps on the struct** — store/clients/clock as fields; construct at app edge.  
4. **Orchestrate once** per method: read → decide (prefer pure) → write → return.  
5. **Split thinking vs shell** (sans-IO for rules; time as a parameter).  
6. **Do not re-export** every private step. “One function per task” is interior.  
7. Prefer **methods over free `pub fn` orchestration** when the job has collaborators or more than one use case.  
8. **Data ownership** — this job writes its tables; others use this API.  
9. **Wire at the app edge** — HTTP/CLI holds the struct and stays thin.  
10. **Good enough** — skip formal repo/UoW/aggregate unless a real pain appears.  

## Do not

- App-wide `controllers/` / `services/` / `repositories/` as the project spine  
- Treat “no layers” as “no public orchestration”  
- Re-export helpers “for tests”  
- Default to free public functions that re-pass the same deps every call  
- Invent Cosmic/Clean folder sets by default  

## Output when designing

1. Job + public **struct** name  
2. Methods (use cases)  
3. Fields / deps  
4. Pure rules vs IO bits  
5. Data owned  
6. What you are **not** adding yet  

Then implement only what was asked, or stop after design if design-only.

## Related

- Review: skeptic **architecture**  
- Always-on: pack `context/AGENTS.md` § Project layout  
