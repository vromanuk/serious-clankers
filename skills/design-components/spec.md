# Design Components

## Intent

Design and grow **job-shaped components** with a **small public face** — prefer **one struct** whose **methods** are use cases that orchestrate once — and a **private interior** using pure-core / shell (sans-IO). Merge of component-based layout and thinking-vs-shell — not a second Cosmic/DDD curriculum and not app-wide horizontal layers.

## Triggers

- **SHOULD** apply when designing a component, introducing a job-shaped module/crate, or replacing a public helper dump with a real face.  
- **SHOULD NOT** replace the full skeptic pipeline for arbitrary review.  
- **SHOULD NOT** force formal repository / unit-of-work / aggregate types for simple jobs.

## Behaviors

### Behavior: One merged model

The agent SHALL treat the component’s public API as preferably **one job struct** whose **methods** are the use cases, holding collaborators as fields when the job has IO or shared deps. The agent SHALL keep multi-step load→decide→save **inside** those methods (or private helpers only they call). The agent SHALL place business rules in **thinking code** (no IO) where they are pure decisions, and IO in thin shell paths. The agent SHALL apply deep-module judgment from `philosophy-of-design.md` when the public face is non-trivial: pull complexity down, hide design decisions, avoid temporal public pipelines and pass-through-only APIs. The agent MUST NOT introduce app-wide `controllers/` / `services/` / `repositories/` as the primary project structure. The agent MUST NOT present Cosmic Python’s full pattern stack as the default. The agent SHOULD prefer methods on a struct over free public orchestration functions that re-pass the same dependencies on every call.

#### Scenario: Stray public helpers

- **GIVEN** a component that exports step helpers callers must sequence  
- **WHEN** designing or repairing  
- **THEN** the agent folds them into methods on a public job struct and keeps steps private

#### Scenario: Free functions re-pass deps

- **GIVEN** several public free functions that each take the same store/clock/client parameters  
- **WHEN** designing or repairing a multi-use-case job  
- **THEN** the agent prefers a struct that holds those collaborators and exposes methods

#### Scenario: Pure rules mixed with IO

- **GIVEN** allocation/business rules that call the database mid-function with no pure core  
- **WHEN** designing  
- **THEN** the agent separates thinking (data in → decision out) from shell (load/save)

#### Scenario: Small job

- **GIVEN** a simple job with one use case  
- **WHEN** designing  
- **THEN** the agent does not invent empty adapter packages or formal UoW without need

#### Scenario: Temporal public pipeline

- **GIVEN** a design that exports public load / validate / save stages callers must order  
- **WHEN** designing  
- **THEN** the agent folds them into a use-case method on the job struct and keeps stages private

## Constraints

### Constraint: Layers ban vs public orchestration

The agent MUST NOT treat “no layered architecture” as “no public orchestration surface.” The public job struct (and its methods) **is** the component face.

### Constraint: Simplicity

The agent MUST prefer the short guide model over pattern catalogs. Optional Cosmic names only when the user asks or a documented pain appears.

<!-- skillet-version: 1.7.0 -->
