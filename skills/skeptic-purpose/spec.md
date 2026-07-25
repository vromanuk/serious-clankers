# Skeptic Purpose

## Intent

Stage 1 of skeptic: judge whether the change matches a real need, sketch structure, flag serious approach bugs, and offer real design alternatives when material forks exist.

## Triggers

- **SHOULD** apply when skeptic runs stage 1, or the user asks only for purpose/design-sense review.
- **SHOULD NOT** apply as a full multi-stage review substitute (use skeptic).

## Behaviors

### Behavior: Real need first

The agent SHALL state the real need in one plain sentence and label it assumed when unverified.

#### Scenario: Unclear request

- **GIVEN** a diff with no stated goal
- **WHEN** reviewing purpose
- **THEN** the agent writes a need sentence marked assumed (or asks if blocked)

### Behavior: Structure before taste

The agent SHALL include plain structure diagrams (L0–L2 as fit) before debating style, and SHALL NOT use Mermaid.

#### Scenario: Multi-component change

- **GIVEN** a change touching several modules
- **WHEN** reviewing purpose
- **THEN** the report includes neighbor/system and in-scope component sketches in plain text

### Behavior: Real alternatives only

The agent SHALL offer LETTER options only for material design forks, each with what/pros/cons/gain/worse-when, and SHALL NOT list “do nothing” as an option.

#### Scenario: Two real designs

- **GIVEN** the approach could be push or pull
- **WHEN** presenting options
- **THEN** at least two real options appear with recommended first and full option fields

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand into pure-core, comment form, or hard-rule checklist work belonging to later stages.

<!-- skillet-version: 1.7.0 -->
