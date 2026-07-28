# Observability

## Intent

Give engineers and agents a concrete contract for **production observability**: structured logs, hierarchical units of work (spans), metrics with safe labels, and panic reporting into the same pipeline. Rust-first (`tracing`, metrics, OpenTelemetry), portable ideas. Teaching and review use `references/guide.md`.

## Triggers

- **SHOULD** apply when adding or changing logging, tracing, metrics, or panic hooks.
- **SHOULD** apply when reviewing whether a change is operable in production.
- **SHOULD** apply when skeptic runs the observability stage.
- **SHOULD NOT** apply as a substitute for full multi-lens review or pure unit-test design alone.

## Behaviors

### Behavior: Useful structured signals

The agent SHALL prefer structured, filterable telemetry over `println!`-style production logging, and SHALL flag telemetry that is useless noise or couples message content to a fixed destination.

#### Scenario: println in request handler

- **GIVEN** a service handler uses `println!` for request outcomes
- **WHEN** reviewing observability
- **THEN** the agent flags it and prefers a tracing/logging facade with fields and configurable export

### Behavior: Spans for units of work

The agent SHALL expect meaningful units of work to be represented as spans with duration and outcome where relevant, fields declared before record, and correct enter/instrument behavior including async and spawn.

#### Scenario: record without declared field

- **GIVEN** code calls `span.record("outcome", …)` without declaring `outcome` at span creation
- **WHEN** reviewing
- **THEN** the agent flags a silent no-op risk and requires declaring the field up front

#### Scenario: tokio::spawn loses parent

- **GIVEN** work is spawned without `.instrument` while parent context matters
- **WHEN** reviewing
- **THEN** the agent flags missing span attachment

### Behavior: Safe metrics labels

The agent SHALL flag metric labels with unbounded or high-cardinality values (e.g. raw user ids) and prefer bounded dimensions.

#### Scenario: user_id label

- **GIVEN** a counter increments with a `user_id` label
- **WHEN** reviewing
- **THEN** the agent flags cardinality risk

## Constraints

### Constraint: Load the guide for depth

For non-trivial write or review of telemetry, the agent MUST load `references/guide.md` rather than relying only on the short SKILL checklist. The agent MUST NOT invent vendor-only APIs as the only option when OpenTelemetry-style export is the local standard.

### Constraint: No secrets in signals

The agent MUST NOT recommend logging secrets or putting secrets in span/metric fields.

<!-- skillet-version: 1.7.0 -->
