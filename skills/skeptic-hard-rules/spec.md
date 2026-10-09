# Skeptic Hard Rules

## Intent

Stage 10 of skeptic: pass/fail absolute bans only, from `references/hard-rules.md` on this skill. No soft taste.

## Triggers

- **SHOULD** apply when skeptic runs stage 10, or the user asks only for hard-rules review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Checklist only

The agent SHALL walk `references/hard-rules.md` against the diff and report only rule-id hits with locators, or `none`. That checklist includes API/interface rules (`HR-api-*`) for public surfaces, IO rules such as `HR-io-timeout` for external/remote calls, and `HR-sleep-in-tests` for sleeps and wall-clock spins in tests.

#### Scenario: Private import

- **GIVEN** a diff imports another component’s private path
- **WHEN** reviewing hard rules
- **THEN** the agent reports `HR-private-import` with path:line and smallest fix

#### Scenario: Bare duration on public wire type

- **GIVEN** a public JSON/API field `"duration": 5000` with no unit in name or structure
- **WHEN** reviewing hard rules
- **THEN** the agent reports `HR-api-bare-quantity` with path:line and smallest fix

#### Scenario: External call without timeout

- **GIVEN** a new or changed outbound HTTP/DB/gRPC (or similar remote IO) with no client timeout, per-request timeout, or wrapping deadline
- **WHEN** reviewing hard rules
- **THEN** the agent reports `HR-io-timeout` with path:line and smallest fix

#### Scenario: Sleep in a test

- **GIVEN** a test sleeps, or spins until wall-clock time passes, before asserting — including a test that calls production code and waits out that code’s real sleep
- **WHEN** reviewing hard rules
- **THEN** the agent reports `HR-sleep-in-tests` as a blocker with path:line and the smallest fix (fake clock, or wait on the event). A comment explaining the sleep is not a fix. Production sleep outside tests is not this rule.

### Behavior: No invented IDs

The agent SHALL NOT invent hard-rule IDs not present in that hard-rules file.

#### Scenario: Soft nit

- **GIVEN** only a naming preference with no rule id
- **WHEN** reviewing hard rules
- **THEN** the agent leaves it out of this section (or routes to comments/naming/conventions), not a fake HR id

## Constraints

### Constraint: Pass/fail only

The agent MUST NOT expand this stage into architecture or pure-core essays.

<!-- skillet-version: 1.7.0 -->
