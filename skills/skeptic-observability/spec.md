# Skeptic Observability

## Intent

Stage 5 of skeptic: judge production observability of the change — structured logs, spans for units of work, metrics label safety, panic reporting, async/thread span attachment. Depth from the `observability` skill.

## Triggers

- **SHOULD** apply when skeptic runs stage 5, or the user asks only for observability/telemetry review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Load observability contract

The agent SHALL load the observability skill guide when the diff touches logging, tracing, metrics, panic hooks, or production request/job paths, and SHALL report findings with path:line or `none` with reason when telemetry is out of scope.

#### Scenario: New HTTP handler without structure

- **GIVEN** a new request handler with only `println!` diagnostics
- **WHEN** reviewing observability
- **THEN** the agent flags unstructured production logging

#### Scenario: Pure rename, no runtime change

- **GIVEN** a rename-only diff with no I/O or telemetry change
- **WHEN** reviewing observability
- **THEN** the agent may report `none` with a short why

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into full architecture redesign or a hard-rule checklist walk (except secret leakage in signals, which may be noted and routed).

<!-- skillet-version: 1.7.0 -->
