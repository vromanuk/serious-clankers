# Skeptic Architecture

## Intent

Stage 2 of skeptic: judge and uphold the pack’s **default project architecture** — component-based layout (job-shaped components, public vs private surface as **use-case orchestration** not stray helpers, nested internals, data ownership, edge composition), dependency direction, and whether types force real contracts at boundaries. The same layout is the default when creating structure, not only when reviewing.

## Triggers

- **SHOULD** apply when skeptic runs stage 2, or the user asks only for architecture/component review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Job-shaped components

The agent SHALL treat job-shaped components (public surface at the component boundary + private implementation, edge composition, data ownership) as the **default** architecture for project layout. The agent SHALL prefer that shape over layer-only layout, SHALL judge the **boundary** (what the root exports vs private modules), and SHALL flag imports past the public surface and shared write free-for-alls with path:line evidence. The agent SHALL treat the public surface as preferably **one job struct** with **use-case methods** (orchestration entrypoints) and SHALL flag shallow bags of public step-helpers, free orchestration functions that re-pass the same deps, or multi-step workflows only in handlers when a component should own them. The agent MUST NOT require folders named `api` or `internal` when the component root already defines a clear public interface (e.g. Rust `mod.rs` / `lib.rs`). The agent MUST NOT treat “no app-wide layers” as forbidding a public job struct on the component. The agent MUST NOT demand Cosmic/DDD ceremony (formal UoW/repository hierarchies) by default. When layout is in scope, the agent SHALL report `components: ok` or concrete layout findings. When the change **adds** structure, the agent SHALL judge it against this default unless the user or existing codebase clearly requires another shape.

#### Scenario: Cross-module private import

- **GIVEN** a call into another component’s internal package
- **WHEN** reviewing architecture
- **THEN** the agent reports a finding with locator and impact

#### Scenario: Layer-only layout for a multi-job change

- **GIVEN** a change that spans multiple business jobs but only uses global controller/service/repository folders
- **WHEN** reviewing architecture
- **THEN** the agent flags missing job-shaped ownership or unclear component boundaries

#### Scenario: Stray public helpers instead of use cases

- **GIVEN** a component root that publicly exports step helpers callers must sequence (e.g. load, validate, write) for one workflow
- **WHEN** reviewing architecture
- **THEN** the agent flags the shallow surface and prefers methods on a public job struct

#### Scenario: Handler owns orchestration

- **GIVEN** an HTTP or CLI handler that performs multi-step load → domain → commit for a job that has (or should have) a component
- **WHEN** reviewing architecture
- **THEN** the agent flags missing component-level orchestration on a public job struct

#### Scenario: Free functions re-pass deps

- **GIVEN** several public free functions that each take the same store or client for one job
- **WHEN** reviewing architecture
- **THEN** the agent prefers a job struct that holds those collaborators and exposes methods

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
