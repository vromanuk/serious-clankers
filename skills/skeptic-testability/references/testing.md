# Testing guidelines

Load when writing or reviewing tests.  
Also: `pure-core.md` (state/invariant ideas); unit craft: skill `unit-tests` → `../unit-tests/references/guide.md`.

Inspired by (portable ideas): *Software Engineering at Google* Ch. 12 Unit Testing — not Java-specific copy.

---

## Goals

1. **Catch real bugs** before users do.  
2. **Raise productivity** — fast feedback, confidence to change code.  
3. **Stay maintainable** — after writing, a test should “just work” until requirements change; failures should point at real bugs with a clear cause.

A suite that breaks on every harmless refactor is a tax, not a safety net.

---

## Size vs scope (rough map)

| Idea | Meaning for us |
|------|----------------|
| **Narrow scope (unit)** | One function, type, or small pure module — usually no network/disk/clock |
| **Broader scope** | Shell, integration, multi-crate, real DuckDB/files — fewer, slower |
| **Rule of thumb** | Most tests **unit/narrow**; minority broader (Google’s ~80/20 is a direction, not a quota) |

Small/narrow tests: fast, deterministic, run constantly. Broad tests: prove boundaries the unit suite cannot.

---

## Maintainable tests

### Strive for unchanging tests

After a test is written, you should rarely touch it unless **behavior/requirements** change.

| Production change | Tests should… |
|-------------------|---------------|
| Pure refactor (same public behavior) | **Not** need edits; if they break, tests were too tied to internals or the change isn’t pure |
| New feature | **Add** tests for new behavior; existing tests stay green |
| Bug fix | **Add** the missing case; existing tests usually unchanged |
| Intentional behavior change | **Update** tests — that cost is real (users may depend on old behavior) |

If every small refactor forces a wall of test edits, the tests are brittle.

### Test via the public contract (not private guts)

- Exercise the unit the way a **caller** would: public API / exported functions / decision function inputs and outputs.  
- Prefer not to make private helpers `pub` only for tests, or assert serialization layout / field order of internal structs unless that **is** the contract.  
- Tests that dig into private methods break on renames and extractions with **no user impact** — classic brittleness.
- **Do not test production helpers** as if they were the unit; cover their behavior through the public surface.

“Public” here means the boundary of the **unit you chose** (function, module, crate API) — not necessarily every `pub` keyword in the language.

### DAMP over DRY (helpers in tests)

Tests optimize **clarity**, not shared lines (**DAMP** = Descriptive And Meaningful Phrases). Prefer slightly longer, readable cases over DRY helpers that hide the point.

| Don’t | Do |
|-------|-----|
| `runHappyPath()` / `createUsers(false, true)` as the whole body | Inputs, action, and expected outcome visible in the test |
| Silent `setUp` values the assert depends on | State values that matter **in the body** (or named overrides) |
| Kitchen-sink `validateEverything(...)` | Assert the one behavior this test owns |

Helpers are fine for **boring** defaults (`newCalculator()`, blank fixtures). They are bad when understanding the case requires opening the helper. Full craft: skill **`unit-tests`**.

### Clarity when tests fail

- One behavior per test (or clear cases in a table).  
- Name for the behavior: “does X when Y”, not “test1”.  
- Assert **outcomes** users care about, not “helper Z was called” (unless the call *is* the contract).  
- Avoid huge setup that hides what is under test.

### Brittle patterns to avoid

| Pattern | Why it hurts |
|---------|----------------|
| Asserting internal structure / private production helpers | Refactors for clarity break tests |
| Scenario helpers / DRY fixtures that hide the case | Unclear failures; helper bugs mask product bugs |
| Over-mocking internal modules | Tests lock implementation, not behavior |
| Order-dependent tests / shared mutable globals | Flakes; hard to run alone |
| Sleeps for “eventual” success | Flakes; prefer fake clocks / pure cores |
| Snapshot of entire huge unstable output without discipline | Constant golden churn |

---

## Property-based tests (when applicable)

**What:** generate many inputs (and sequences); assert **invariants** that must always hold; on failure, shrink to a small counterexample.

**When useful**

| Fit | Examples |
|-----|----------|
| Pure functions with a wide input space | Parsers, window math, serializers round-trip, codecs |
| Stateful cores | `step(state, event)` — random legal event sequences; assert invariants after every step (see `pure-core.md`) |
| “Never” / “always” rules | Never commit past unpersisted work; checksum; monotonic counters |

**When not**

| Poor fit | Prefer |
|----------|--------|
| Thin shell over real S3/DB | Few integration tests |
| One obvious branch | A few explicit cases |
| UI layout pixel-perfect | Snapshot or manual |

**Rust:** `proptest` (or similar) for generators; keep properties next to the pure module. Fixed seeds in CI when you need reproducibility.

**Relation to given/expected:** design headers and table tests nail **named** contracts; property tests hunt **holes** the examples missed. Use both when the domain is rich.

---

## Snapshot / golden tests (when applicable)

**What:** compare output (string, JSON, bytes, rendered HTML) to a checked-in **golden** file; failures show a diff; update goldens deliberately when behavior changes on purpose.

**When useful**

| Fit | Examples |
|-----|----------|
| Stable serialized form is the product | SQL string builders, config render, error message catalogs (if stable by design) |
| Complex structured output | `run_meta`-like JSON shape, HTML report fragments (if you want lock-in) |
| Regression on formatting that is hard to assert field-by-field | Multi-line diagnostics |

**When not / careful**

| Risk | Mitigate |
|------|----------|
| Goldens encode noise (timestamps, absolute paths, host) | Normalize before compare; inject fixed clocks/paths in tests |
| Every tiny UI tweak updates dozens of files | Snapshot only the **stable contract surface**, not everything |
| People “accept” diffs without reading | Review golden updates like production code |

**Rust:** `insta` is common; or hand-written expected strings for small cases (many of our bench tests already do substring/exact string asserts — that’s a lightweight snapshot).

**Property vs snapshot:** properties check laws across many inputs; snapshots lock **one** concrete output. Different tools.

---

## Thinking code vs shell (tests)

| Code | Tests |
|------|--------|
| Thinking (pure) | Unit: fixed inputs/outputs; property/invariant when space is big; snapshot/golden when a stable blob is the contract; **no** network/disk/clock |
| Shell (IO) | Few integration tests through the real boundary when practical |
| Don’t | Mock forest of internal helpers instead of fixing the pure/shell split |

Time: pass `now` into pure code; tests use fixed instants.

---

## Mix of tests (practical)

```text
  Many:   pure unit + table cases from design headers
  When fit: property/sequence + invariants (laws, wide space, state machines)
  When fit: snapshot/golden (stable serialized/rendered blob is the contract)
  Few:    integration (files, DuckDB, network)
  Rare:   full-system / manual (document in PR Testing section)
```

Do not treat property tests as the only “extra” tool after unit tests. If the
product is a concrete string/JSON/SQL shape, a disciplined snapshot often beats a
vague property. If the product is a law over many inputs, prefer properties.

For **benches**: unit-test SQL builders and pure window math; don’t replace A/B experiments with unit tests — different job.

---

## PR / integration hygiene (portable)

| Idea | Practice |
|------|----------|
| **How was it validated?** | Describe how the change was or will be validated. “No testing required” is not enough — if automation is hard, say the **manual/post-merge** check (what, where). |
| **Unit vs real-world boundary** | Anything that hits network, real DB, real filesystem, live services, or binds ports is **integration** — keep those few and separate from pure unit tests of thinking code. |
| **Don’t mix suites** | Don’t bury pure unit cases inside a suite that only exists to do real IO (and vice versa). |
| **Independent tests** | Each test runs alone and in any order; no relying on another test’s side effects. |
| **Avoid flaky timing** | Prefer fake clocks / pure cores over `sleep`. If a timing wait is truly needed, name and comment why, and use coarse granularity. |
| **Obvious test code** | Prefer clear, slightly longer tests over clever test-only frameworks that hide the behavior under test. |
| **Test-only hooks stay narrow** | Don’t widen the public API only so tests can poke internals; keep test seams small and hard to misuse in prod. |
| **Public surface deserves tests** | If something is part of the unit’s real contract for callers, prefer an explicit test; if you wouldn’t test it, question whether it should be public. |
| **Prefer tests over prose-only rules** | Encode what you can in automated tests and linters. |

---

## Checklist when adding tests

1. Does this lock a **user-visible or public** contract, or only a private rename / production helper? Prefer the former.  
2. Will this test still make sense after a pure refactor?  
3. On failure, is the next step obvious **without opening test helpers**?  
4. **DAMP?** Important inputs and expected outcomes visible in the body?  
5. Wide input space or long-lived state → consider **property / sequence + invariant**.  
6. Stable complex blob → consider **snapshot** with normalization.  
7. New behavior → new test (`HR-new-behavior-no-test` for thinking code).

---

## Sources

- SE at Google Ch. 12 (unit testing maintainability, public API, unchanging tests) — portable ideas only  
- `pure-core.md`; composition depth: `skeptic-conventions/references/write-code.md`; comments: `skeptic-comments`  
