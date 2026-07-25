---
name: unit-tests
description: >
  Unit-test guidelines for writing and reviewing maintainable unit tests
  (Google SE Ch.12-style: unchanging tests, public API, state not interactions,
  behaviors not methods, DAMP over DRY). Use when adding/changing unit tests,
  reviewing test quality or brittleness, or when skeptic/testability stage needs
  unit-test depth. Not for pure integration/E2E strategy alone.
---

# Unit tests

Write and judge **unit tests** so they raise productivity: fast feedback, real bugs on failure, almost never rewritten unless product behavior changes.

**Full rules + examples:** load `references/guide.md` (required for non-trivial work).  
Samples in the guide are mostly **Java** (from the source chapter) — principles apply in any language; the code is reference material, not a stack requirement.

## First rules (always)

1. **Unchanging tests** — after write, touch only when product behavior changes (not pure refactor / new feature / bugfix on *other* cases).  
2. **Public API of the unit** — exercise like a real caller; not private helpers / serialization guts.  
3. **State, not interactions** — assert resulting state; don’t verify every collaborator call (mocks that record all calls → brittle).  
4. **Behaviors, not methods** — one behavior per test; name after the behavior; structure Given/When/Then (or arrange/act/assert).  
5. **Complete and concise** — relevant inputs and outcomes visible; no clutter; no hidden setup that hides the point.  
6. **No clever logic in tests** — straight-line code; tolerate duplication when it clarifies; **DAMP over DRY**.  
7. **Clear failure messages** — expected vs actual + context; use good assertion libraries.  

## Do (write)

- Aim ~**80% unit** / ~20% broader (rule of thumb, not quota).  
- Unit ≈ narrow scope (function/type/module); prefer **small** (fast, deterministic).  
- Public API tests double as documentation.  
- New feature / bug fix → **add** tests; don’t rewrite the whole suite.  
- Prefer real collaborators when fast and deterministic; mock only at true boundaries.  
- Shared helpers: keep tests readable (DAMP); give setup explicit names; don’t hide important values in shared fixtures.  
- Cross-suite test infrastructure = its own product (with its own tests); prefer standard org libraries.  

## Do (review)

Flag with path:line:

| Smell | Why |
|-------|-----|
| Tests private methods / internal serialization | Brittle on pure refactor |
| `verify(mock).foo()` as the main assert | Interaction test; prefer state |
| One test covering many unrelated behaviors | Hard failures; split by behavior |
| Name is `testFoo` / method-shaped only | Rename after behavior |
| Loops/conditionals/helpers that reimplement production | Logic in tests hides bugs |
| Shared `setUp` values tests depend on silently | Unclear; make relevant values local or named helpers |
| Over-mocked internals | Locks implementation |
| Failure message is only “expected true” | Improve asserts |

## Scope

- **In:** unit tests, brittleness, clarity, naming, structure, DAMP/DRY, state vs interaction.  
- **Out:** full integration/E2E design alone; property/snapshot *mechanics* (see `skeptic/references/testing.md` for those tools — they still obey unchanging/public-API rules).  

## Related

- Skeptic: `skeptic-testability` loads this for unit depth; `skeptic/references/testing.md` for property/snapshot shapes.  
- Local review pack: `review-testability` loads the same skill under `~/agents/skills/unit-tests/`.  
