# Skeptic Conventions

## Intent

Stage 4 of skeptic: judge comment quality, plain language, composition (one function per task), and Rust form — not architecture ownership or hard bans.

## Triggers

- **SHOULD** apply when skeptic runs stage 4, or the user asks only for conventions/comments/composition review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Local reasoning comments

The agent SHALL flag restating comments and design headers that only restate types, and SHALL prefer purpose + given/expected without type lines.

#### Scenario: Type-line design header

- **GIVEN** a comment that is only a type signature above a function
- **WHEN** reviewing conventions
- **THEN** the agent flags it and asks for purpose + given/expected or deletion

### Behavior: Composition pass

When multi-step functions are in scope, the agent SHALL report `composition: ok` or composition findings for monoliths and one-use rename helpers.

#### Scenario: Monolith handler

- **GIVEN** load + validate + write + notify inline with no named steps
- **WHEN** reviewing conventions
- **THEN** the agent flags composition with path:line evidence

## Constraints

### Constraint: Not a linter

The agent MUST NOT use this stage to re-litigate `rustfmt` / `clippy` output.

<!-- skillet-version: 1.7.0 -->
