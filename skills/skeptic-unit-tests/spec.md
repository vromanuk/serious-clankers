# Skeptic Unit Tests

## Intent

Stage 4 of skeptic: always judge **unit-test craft** on the scoped change (unchanging tests, public API of the unit, state not interactions, behaviors not methods, DAMP over DRY). Loads the `unit-tests` skill. Not pure-core placement or property/snapshot strategy alone.

## Triggers

- **SHOULD** apply when skeptic runs stage 4 (always in the pipeline), or the user asks only for unit-test quality review.  
- **SHOULD NOT** replace the full skeptic pipeline.  
- **SHOULD NOT** skip when the diff “looks like no tests” — still run and report `none` with reason if nothing applies.

## Behaviors

### Behavior: Always run unit-test craft

The agent SHALL apply the unit-tests skill checklist to new/changed tests and to production changes that need unit coverage. The agent SHALL flag brittle, helper-hidden, private-API, or interaction-only unit tests with path:line evidence. When no tests and no new pure behavior are in scope, the agent SHALL report `none` (or `unit-tests: ok`) with a one-line reason rather than omitting the stage.

#### Scenario: Brittle private helper test

- **GIVEN** a new test that only calls a private production helper
- **WHEN** reviewing unit tests
- **THEN** the agent flags it and prefers a public-API outcome assert

#### Scenario: No tests in scope

- **GIVEN** a pure comment/doc rename with no test or pure-behavior change
- **WHEN** reviewing unit tests
- **THEN** the agent reports none with a short why, not a skipped section

#### Scenario: New pure behavior without unit test

- **GIVEN** thinking-code decision logic added with no unit test on the public contract
- **WHEN** reviewing unit tests
- **THEN** the agent flags the coverage gap (and may note `HR-new-behavior-no-test` for hard-rules)

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into pure-core extraction essays, full observability, or hard-rule ID walks. Property/snapshot shape choice stays testability unless the unit test itself is brittle.

<!-- skillet-version: 1.7.0 -->
