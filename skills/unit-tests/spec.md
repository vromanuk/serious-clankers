# Unit Tests

## Intent

Give engineers and agents a concrete contract for writing and reviewing **unit tests**: maintainable, mostly unchanging, public-API/state-based, behavior-named, DAMP, with clear failures. Teaching and review use the detailed guide in `references/guide.md`.

## Triggers

- **SHOULD** apply when adding, changing, or reviewing unit tests.
- **SHOULD** apply when judging brittleness, mock-heavy tests, or unclear test failures.
- **SHOULD** apply when skeptic-testability (or a local review testability stage) needs unit-test depth.
- **SHOULD NOT** apply as the only guide for pure integration/E2E strategy (broader tests still exist; this skill is unit-focused).
- **SHOULD NOT** replace product implementation skills when the user only wants production code with no test discussion.

## Behaviors

### Behavior: Unchanging and public API (do not test production helpers)

The agent SHALL prefer tests that need no edit under pure refactor/new feature/bug fix of other cases, and SHALL exercise the unit through its public API rather than private production helpers or internal serialization.

#### Scenario: Tests private validator

- **GIVEN** a test calls a private `isValid` and asserts serialization format of DB rows
- **WHEN** reviewing or rewriting
- **THEN** the agent flags brittleness and shows a public-API state-based alternative (e.g. balances after processTransaction)

#### Scenario: pub only for tests

- **GIVEN** a private helper is made `pub` / `#[cfg(test)]` only so unit tests can call it
- **WHEN** reviewing
- **THEN** the agent flags it and prefers covering the behavior via the real public surface (or extracting a real unit if the helper *is* the product)

### Behavior: State over interaction

The agent SHALL prefer asserting system state after the act over verifying exact collaborator call sequences, and SHALL flag mock verify-only tests when a state assert is possible.

#### Scenario: verify(database).put

- **GIVEN** the only assert is `verify(database).put(...)`
- **WHEN** reviewing
- **THEN** the agent recommends asserting observable state (e.g. user exists) and notes false pass/fail risks of interaction checks

### Behavior: Behaviors, structure, naming

The agent SHALL prefer one behavior per test, Given/When/Then (or arrange/act/assert) structure, and names that describe the behavior, not the production method name alone.

#### Scenario: testProcessTransaction

- **GIVEN** a single test named after a method covering several outcomes
- **WHEN** advising
- **THEN** the agent suggests splitting into behavior-named cases (e.g. sufficient balance vs insufficient)

### Behavior: Clarity and DAMP (do not hide cases in test helpers)

The agent SHALL prefer complete, concise, straight-line tests with clear failure messages, and **DAMP over DRY**: descriptive local values and only boring/default helpers — never helpers that hide the behavior under test or reimplement production logic.

#### Scenario: Shared setUp hides amounts

- **GIVEN** transfer amounts only appear in setUp and the test body has no numbers
- **WHEN** reviewing
- **THEN** the agent flags unclarity and asks for relevant values in the test or named helpers with explicit overrides

#### Scenario: Scenario helper is the whole test

- **GIVEN** a test body is only `runHappyPath()` / `createUsers(false, true)` + `validateEverything(...)` with no visible inputs/outcomes
- **WHEN** reviewing or writing
- **THEN** the agent flags incomplete DAMP and rewrites so the case (inputs, action, expected state) is readable without opening helpers

## Constraints

### Constraint: Load the guide for depth

For non-trivial write or review of unit tests, the agent MUST load `references/guide.md` rather than relying only on the short SKILL checklist.

### Constraint: No fake coverage via brittle tests

The agent MUST NOT recommend testing private implementation details solely to raise coverage metrics when public-API tests can express the contract.

### Constraint: Prefer DAMP over DRY for test code

The agent MUST NOT extract shared test helpers that hide behavior-critical inputs or expected outcomes solely to shorten tests. Helpers MAY hide only uninteresting construction defaults.

<!-- skillet-version: 1.7.0 -->
