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

### Behavior: Unchanging and public API

The agent SHALL prefer tests that need no edit under pure refactor/new feature/bugfix of other cases, and SHALL exercise the unit through its public API rather than private helpers or internal serialization.

#### Scenario: Tests private validator

- **GIVEN** a test calls a private `isValid` and asserts serialization format of DB rows
- **WHEN** reviewing or rewriting
- **THEN** the agent flags brittleness and shows a public-API state-based alternative (e.g. balances after processTransaction)

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

### Behavior: Clarity and DAMP

The agent SHALL prefer complete, concise, straight-line tests with clear failure messages, and DAMP sharing (descriptive local values / explicit helpers) over DRY that hides meaning.

#### Scenario: Shared setUp hides amounts

- **GIVEN** transfer amounts only appear in setUp and the test body has no numbers
- **WHEN** reviewing
- **THEN** the agent flags unclarity and asks for relevant values in the test or named helpers with explicit overrides

## Constraints

### Constraint: Load the guide for depth

For non-trivial write or review of unit tests, the agent MUST load `references/guide.md` rather than relying only on the short SKILL checklist.

### Constraint: No fake coverage via brittle tests

The agent MUST NOT recommend testing private implementation details solely to raise coverage metrics when public-API tests can express the contract.

<!-- skillet-version: 1.7.0 -->
