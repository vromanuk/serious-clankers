# Skeptic Naming

## Intent

Stage 5 of skeptic: judge whether names fully say what items are or do, without excess length or unclear abbreviations. Ideas from Google C++ Naming; spelling style follows the project.

## Triggers

- **SHOULD** apply when skeptic runs stage 5, or the user asks only for naming review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Clear names for the scope

The agent SHALL flag names that fail to say their purpose for how widely they are used, and names that are overlong or only repeat local context. The agent SHALL prefer clear names matched to scope and SHALL not force another language’s spelling style over the project’s style.

#### Scenario: Cryptic public name

- **GIVEN** a public function named `proc` or `cstmr_id` with no clear domain abbreviation
- **WHEN** reviewing naming
- **THEN** the agent flags it and suggests a clearer name

#### Scenario: Overlong local

- **GIVEN** a loop local named `total_number_of_foo_errors_in_this_function` next to a clear `foos` slice
- **WHEN** reviewing naming
- **THEN** the agent flags unnecessary length for that scope

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into full comment review, composition redesign, or hard-rule walks.

<!-- skillet-version: 1.7.0 -->
