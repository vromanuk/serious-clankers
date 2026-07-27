# Skeptic Architecture

## Intent

Stage 2 of skeptic: judge and uphold the pack’s **default project architecture** — component-based layout (job-shaped components, public vs private surface, nested internals, data ownership, edge composition), dependency direction, and whether types force real contracts at boundaries. The same layout is the default when creating structure, not only when reviewing.

## Triggers

- **SHOULD** apply when skeptic runs stage 2, or the user asks only for architecture/component review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Job-shaped components

The agent SHALL treat job-shaped components (public surface + private interior, edge composition, data ownership) as the **default** architecture for project layout. The agent SHALL prefer that shape over layer-only layout, SHALL distinguish public surface from private interior, and SHALL flag private-surface imports and shared write free-for-alls with path:line evidence. When layout is in scope, the agent SHALL report `components: ok` or concrete layout findings. When the change **adds** structure, the agent SHALL judge it against this default unless the user or existing codebase clearly requires another shape.

#### Scenario: Cross-module private import

- **GIVEN** a call into another component’s internal package
- **WHEN** reviewing architecture
- **THEN** the agent reports a finding with locator and impact

#### Scenario: Layer-only layout for a multi-job change

- **GIVEN** a change that spans multiple business jobs but only uses global controller/service/repository folders
- **WHEN** reviewing architecture
- **THEN** the agent flags missing job-shaped ownership or unclear component boundaries

### Behavior: Type-driven boundaries

When APIs or shared results change, the agent SHALL check whether callers can forget a real decision, and SHALL report `type-driven: ok` or type-driven findings.

#### Scenario: Empty list as silent success

- **GIVEN** success type is a bare list while product forbids empty
- **WHEN** reviewing architecture
- **THEN** the agent flags the boundary and prefers a type that forces the contract

## Constraints

### Constraint: Stage boundary

The agent MUST NOT turn this stage into pure-core essays, comment nits, or a full hard-rule checklist walk.

<!-- skillet-version: 1.7.0 -->
