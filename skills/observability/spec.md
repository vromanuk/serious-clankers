# Observability

## Intent

Give engineers and agents a concrete contract for **production observability**: structured logs, hierarchical units of work (spans), metrics with safe labels, panic reporting into the same pipeline, and alert rules that notify a human only for significant events. Rust-first (`tracing`, metrics, OpenTelemetry), Prometheus-style alerts, portable ideas. Teaching and review use `references/guide.md`; alert rules use `references/alerts.md`.

## Triggers

- **SHOULD** apply when adding or changing logging, tracing, metrics, or panic hooks.
- **SHOULD** apply when adding, changing, or reviewing alert rules.
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

### Behavior: Alert rules judged on four properties

The agent SHALL judge each new or changed alert rule on precision, recall, detection time, and reset time, and SHALL state how many alerts fire for one incident given the labels left after aggregation. The agent SHALL prefer symptoms users feel over causes, ratios over raw counts, and window length over a long `for:` duration as the noise filter, and SHALL flag signals that can disappear without a missing-data alert.

#### Scenario: long for on a short window

- **GIVEN** an alert on a 1-minute error ratio with `for: 1h`
- **WHEN** reviewing
- **THEN** the agent flags slow detection that does not scale with severity and missed errors that come and go, and suggests a longer window or a long and short window together

#### Scenario: service-level sum hides one bad pod

- **GIVEN** an error ratio summed across 20 pods, where pods hold their own connection pools
- **WHEN** reviewing
- **THEN** the agent notes that one pod failing completely may stay under the threshold and suggests a per-pod or pod-versus-service alert at ticket severity, suppressed while the service-level alert fires

#### Scenario: unaggregated per-pod alert

- **GIVEN** an error alert with no aggregation on a 20-replica service
- **WHEN** reviewing
- **THEN** the agent flags up to 20 alerts for one incident and the alert identity changing on every pod restart

#### Scenario: ratio averaged across pods

- **GIVEN** an alert on `avg()` of per-pod error ratios
- **WHEN** reviewing
- **THEN** the agent flags the math and prefers the sum of failures divided by the sum of requests

## Constraints

### Constraint: Load the guide for depth

For non-trivial write or review of telemetry, the agent MUST load `references/guide.md` rather than relying only on the short SKILL checklist. For alert rules, the agent MUST load `references/alerts.md`. The agent MUST NOT invent vendor-only APIs as the only option when OpenTelemetry-style export is the local standard.

### Constraint: No secrets in signals

The agent MUST NOT recommend logging secrets or putting secrets in span/metric fields.

<!-- skillet-version: 1.7.0 -->
