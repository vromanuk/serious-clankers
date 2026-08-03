---
name: unit-tests
description: >
  Unit-test guidelines for writing and reviewing maintainable unit tests
  (Google SE Ch.12-style: unchanging tests, public API, state not interactions,
  behaviors not methods, DAMP over DRY — do not test production helpers; do not
  hide the case behind test helpers). Use when adding/changing unit tests,
  reviewing test quality or brittleness, or when skeptic/testability stage needs
  unit-test depth. Not for pure integration/E2E strategy alone.
---

# Unit tests

Write and judge **unit tests** so they raise productivity: fast feedback, real bugs on failure, almost never rewritten unless product behavior changes.

**Full rules + examples:** load `references/guide.md` (required for non-trivial work).  
Samples in the guide are mostly **Java** (from the source chapter) — principles apply in any language; the code is reference material, not a stack requirement.

## First rules (always)

1. **Unchanging tests** — after write, touch only when product behavior changes (not pure refactor / new feature / bug fix on *other* cases).  
2. **Public API of the unit** — exercise like a real caller; **do not test private production helpers** / serialization guts.  
3. **State, not interactions** — assert resulting state; don’t verify every collaborator call (mocks that record all calls → brittle).  
4. **Behaviors, not methods** — one behavior per test; name after the behavior; structure Given/When/Then (or arrange/act/assert).  
5. **Complete and concise** — relevant inputs and outcomes visible; no clutter; no hidden setup that hides the point.  
6. **DAMP over DRY** — tests optimize **clarity**, not shared lines. Straight-line; tolerate duplication when it clarifies; **do not bury the case in test helpers**.  
7. **Clear failure messages** — expected vs actual + context; use good assertion libraries.  

## Helpers: two distinct anti-patterns (highlight)

| Don’t | Why | Do instead |
|-------|-----|------------|
| **Test production helpers** (private/`pub(crate)` only for tests, `isValid`, internal serialize) | Brittle on pure refactor; users never call them | Exercise the **public unit surface**; assert **outcomes/state** |
| **Hide the case in test helpers** (`createUsers(false,true)`, `runHappyPath()`, shared `setUp` values the assert depends on) | Reader can’t see the behavior; helper bugs hide product bugs | **DAMP:** inputs, action, expected result visible in the test body. Helpers only for **boring** defaults (constructors, blank fixtures) with **explicit overrides** for what this case cares about |

**Rule of thumb:** if understanding the test requires opening a helper (or reading `setUp` hundreds of lines away), the helper is too clever or hides too much. If the only way to cover a branch is calling a private helper, the pure surface is wrong or the case belongs on the public API.

## Do (write)

- Aim ~**80% unit** / ~20% broader (rule of thumb, not quota).  
- Unit ≈ narrow scope (function/type/module); prefer **small** (fast, deterministic).  
- Public API tests double as documentation.  
- New feature / bug fix → **add** tests; don’t rewrite the whole suite.  
- Prefer real collaborators when fast and deterministic; mock only at true boundaries.  
- Shared test code: DAMP first — name setup so intent is obvious; leave behavior-critical values in the body.  
- Cross-suite test infrastructure = its own product (with its own tests); prefer standard org libraries.  
- E2E/integration: same DAMP idea — prefer a readable case over a shared “remote demo” helper that rewrites or hides the scenario.  

## Do (review)

Flag with path:line:

| Smell | Why |
|-------|-----|
| Tests private methods / production helpers / internal serialization | Brittle on pure refactor; not the public contract |
| `#[cfg(test)]` / `pub` only so tests can poke helpers | Wrong surface; lift contract or test through callers |
| Shared fixture / helper hides the inputs or expected outcome | Incomplete; DAMP — put the case in the body |
| `runScenario()` / `assertEverything()` used as the whole test | Hides behavior; split or inline |
| Loops/conditionals/helpers that reimplement production | Logic in tests hides bugs |
| `verify(mock).foo()` as the main assert | Interaction test; prefer state |
| One test covering many unrelated behaviors | Hard failures; split by behavior |
| Name is `testFoo` / method-shaped only | Rename after behavior |
| Shared `setUp` values tests depend on silently | Unclear; make relevant values local or named overrides |
| Over-mocked internals | Locks implementation |
| Failure message is only “expected true” | Improve asserts |

## Scope

- **In:** unit tests, brittleness, clarity, naming, structure, **DAMP/DRY**, helpers (production *and* test), state vs interaction.  
- **Out:** full integration/E2E design alone; property/snapshot *mechanics* (see `skeptic-testability/references/testing.md` — they still obey unchanging/public-API/**DAMP** rules).  

## Related

- Skeptic: `skeptic-testability` loads this for unit depth; its `references/testing.md` for property/snapshot shapes.  
- Local review pack: `review-testability` loads the same skill under `~/agents/skills/unit-tests/`.  
