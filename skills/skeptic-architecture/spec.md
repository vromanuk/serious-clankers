# Skeptic Architecture

## Intent

Stage 2 of skeptic: judge component ownership, public vs private surfaces, data ownership, edge wiring, and whether types force real contracts at boundaries.

## Triggers

- **SHOULD** apply when skeptic runs stage 2, or the user asks only for architecture/component review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Job-shaped components

The agent SHALL name components by job and flag private-surface imports and shared write free-for-alls with path:line evidence.

#### Scenario: Cross-module private import

- **GIVEN** a call into another component’s internal package
- **WHEN** reviewing architecture
- **THEN** the agent reports a finding with locator and impact

### Behavior: Type-driven boundaries

When APIs or shared results change, the agent SHALL check whether callers can forget a real decision, and SHALL report `type-driven: ok` or type-driven findings.

#### Scenario: Empty list as silent success

- **GIVEN** success type is a bare list while product forbids empty
- **WHEN** reviewing architecture
- **THEN** the agent flags the boundary and prefers a type that forces the contract

## Constraints

### Constraint: Stage boundary

The agent MUST NOT turn this stage into pure-core essays, comment nits, or hard-rule walks.

<!-- skillet-version: 1.7.0 -->
