---
name: skeptic-unit-tests
description: >
  Stage 4 of skeptic: unit-test craft on the scoped diff — unchanging tests,
  public API of the unit, state not interactions, behaviors not methods, DAMP
  over DRY. Always runs when skeptic runs. Use when skeptic runs or the user
  asks for unit-test quality review only. Loads the unit-tests skill for depth.
  Not for pure-core placement alone (→ testability) or property/snapshot
  mechanics alone (→ testability).
---

# Skeptic unit-tests

**Question:** Are unit tests on this change maintainable — unchanging, public-API, state-based, behavior-named, DAMP — or brittle / helper-hidden / interaction-locked?

This stage **always runs** in the skeptic pipeline. It is not optional depth under testability.

## Load (required when tests or new pure behavior are in scope; still load for the checklist when they might be)

1. `../unit-tests/SKILL.md` — first rules + review smells  
2. `../unit-tests/references/guide.md` — full depth (non-trivial test diffs)  

## Do

1. Apply the **unit-tests** skill review checklist to **new or changed** tests and to production changes that **should** have unit coverage.  
2. Flag (with path:line):  
   - tests of private / production helpers / serialization guts  
   - interaction-only asserts (`verify(mock)…` as the main check)  
   - multi-behavior tests; method-shaped names (`testFoo`)  
   - DAMP failures: case buried in helpers / silent `setUp` / `runScenario` as the whole test  
   - logic in tests that reimplements production  
   - over-mocked internals; weak failure messages  
3. Flag **missing** unit coverage when thinking-code / pure contracts gained behavior and no public-API unit test was added (overlap with testability — report craft + gap here; pure-vs-shell placement stays stage 3).  
4. When tests are in scope: `unit-tests: ok` (one line) or findings.  
5. When **no** test files and **no** new pure behavior in the diff: `none` with one-line why (still run the stage — do not skip silently).  

## Do not

- Pure vs shell / extract decisions (→ testability) unless needed to explain a bad test surface  
- Property vs snapshot **mechanics** as the main pass (→ testability `testing.md`) — still flag if a unit test violates unchanging/public-API/DAMP  
- Integration/E2E strategy alone  
- Comment/naming polish of production code (→ comments / naming)  
- Full hard-rule walk (→ hard-rules); `HR-new-behavior-no-test` may be noted and left to stage 9  

## Output section title

`## 4. Unit tests`
