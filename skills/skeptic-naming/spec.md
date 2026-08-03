# Skeptic Naming

## Intent

Stage 7 of skeptic: judge whether names fully say what items are or do, without excess length or unclear abbreviations. General names from Google C++ Naming; **functions** use a stronger bar (actions = verb phrases; bools = `is`/`has`/`can` predicates); **public/API surface** names use a stronger bar from The API Book (units in names, concrete verbs, pairs match, naming matches type). Spelling style follows the project.

## Triggers

- **SHOULD** apply when skeptic runs stage 7, or the user asks only for naming review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Clear names for the scope

The agent SHALL flag names that fail to say their purpose for how widely they are used, and names that are overlong or only repeat local context. The agent SHALL prefer clear names matched to scope and SHALL not force another language’s spelling style over the project’s style. For functions/methods, the agent SHALL apply stronger rules: work functions named as verb phrases (not bare nouns or amoeba verbs alone); boolean returns named as affirmative predicates (`is_` / `has_` / `can_` or clear equivalents). For public/API surfaces, the agent SHALL also apply stronger rules: explicit side effects, units/standards in names, concrete names (not bare get/process), naming that matches type kind, matching pair words, and no double-negative flags.

#### Scenario: Cryptic public name

- **GIVEN** a public function named `proc` or `cstmr_id` with no clear domain abbreviation
- **WHEN** reviewing naming
- **THEN** the agent flags it and suggests a clearer name

#### Scenario: Overlong local

- **GIVEN** a loop local named `total_number_of_foo_errors_in_this_function` next to a clear `foos` slice
- **WHEN** reviewing naming
- **THEN** the agent flags unnecessary length for that scope

#### Scenario: Bare duration field

- **GIVEN** a public JSON field `"duration": 5000` with no unit
- **WHEN** reviewing naming
- **THEN** the agent flags it and prefers `duration_ms` or an explicit unit shape

#### Scenario: get_time ambiguity

- **GIVEN** a public method `order.get_time()` when several times exist for an order
- **WHEN** reviewing naming
- **THEN** the agent flags it and prefers a concrete name (e.g. estimated delivery time)

#### Scenario: Action named as a noun

- **GIVEN** a function `profile_credentials()` that loads credentials from disk
- **WHEN** reviewing naming
- **THEN** the agent flags it and prefers a verb phrase (e.g. `load_profile_credentials`)

#### Scenario: Bool without predicate form

- **GIVEN** a function `ready(job) -> bool` or `check_valid(x) -> bool`
- **WHEN** reviewing naming
- **THEN** the agent flags it and prefers an affirmative predicate (e.g. `is_ready`, `is_valid`)

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into full comment review, composition redesign, or hard-rule walks.

<!-- skillet-version: 1.7.0 -->
