# Skeptic Testability

## Intent

Stage 3 of skeptic: judge whether decisions are pure and testable, whether IO stays at the edge, and whether new contracts have the right test shape (table / property / snapshot). Unit-test craft (DAMP, unchanging, public API) is stage 4 and always runs separately.

## Triggers

- **SHOULD** apply when skeptic runs stage 3, or the user asks only for testability / pure-core review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline or replace stage 4 unit-tests.

## Behaviors

### Behavior: Thinking vs shell

The agent SHALL flag business rules mixed into IO/clock/env paths and prefer decisions as data returned to a thin shell.

#### Scenario: Policy mid-handler

- **GIVEN** skip/error policy computed between DB calls with no pure helper
- **WHEN** reviewing testability
- **THEN** the agent reports a finding with path:line and a smallest extract-oriented fix

### Behavior: Right test shape

When pure contracts gain behavior, the agent SHALL prefer table, property, or snapshot to match the contract — not property by default.

#### Scenario: Stable report blob

- **GIVEN** the contract is a multi-line diagnostic report
- **WHEN** reviewing tests
- **THEN** the agent prefers snapshot/golden (with normalization) over weak generic properties

## Constraints

### Constraint: Stage boundary

The agent MUST NOT turn this stage into architecture redesign, hard-rule walks, or a full unit-test craft review (that is stage 4).

<!-- skillet-version: 1.7.0 -->
