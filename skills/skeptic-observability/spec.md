# Skeptic Observability

## Intent

Stage 6 of skeptic: judge production observability of the change — structured logs, spans for units of work, metrics label safety, panic reporting, async/thread span attachment, and alert rules. Depth from the `observability` skill.

## Triggers

- **SHOULD** apply when skeptic runs stage 6, or the user asks only for observability/telemetry/alerting review.
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

### Behavior: Review alert rules

The agent SHALL load the observability alerts reference when the diff adds or changes alert rules, SHALL give one line per alert on precision, recall, detection time, reset time, and how many alerts fire for one incident, and SHALL report findings with path:line.

#### Scenario: Threshold steps that all page

- **GIVEN** four alerts on the same 5-minute error ratio at 5%, 10%, 25%, and 50%, all paging
- **WHEN** reviewing observability
- **THEN** the agent notes that a full outage fires all four at once and suggests one paging rule on short windows and one ticket rule on long windows, or suppressing the lower steps

#### Scenario: Summed error ratio across pods

- **GIVEN** a new alert on an error ratio summed across pods, where pods can fail on their own
- **WHEN** reviewing observability
- **THEN** the agent states that it fires once per service, notes the one-bad-pod gap, and suggests a per-pod alert at ticket severity

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into full architecture redesign, a hard-rule checklist walk (except secret leakage in signals, which may be noted and routed), or alert routing and SLO design beyond a short note.

<!-- skillet-version: 1.7.0 -->
