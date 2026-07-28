# Skeptic Conventions

## Intent

Stage 7 of skeptic: judge plain words in code, one function per task, and clear Rust style — not comments, naming, architecture, or hard bans.

## Triggers

- **SHOULD** apply when skeptic runs stage 7, or the user asks only for composition or Rust style review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Composition pass

When multi-step functions are in scope, the agent SHALL report `composition: ok` or composition findings for monoliths and one-use rename helpers.

#### Scenario: Monolith handler

- **GIVEN** load + validate + write + notify inline with no named steps
- **WHEN** reviewing conventions
- **THEN** the agent flags composition with path:line evidence

### Behavior: Easy to read

The agent SHALL flag clever but hard-to-follow code when a simpler shape would read better, unless a real measured hot path has a short local why.

#### Scenario: Clever one-liner

- **GIVEN** a dense one-liner that obscures a simple branch
- **WHEN** reviewing conventions
- **THEN** the agent flags it and prefers a plainer structure unless the hot path is documented

## Constraints

### Constraint: Not a linter

The agent MUST NOT use this stage to re-litigate `rustfmt` / `clippy` output.

### Constraint: Stage boundary

The agent MUST NOT expand this stage into full comment or naming review.

<!-- skillet-version: 1.7.0 -->
