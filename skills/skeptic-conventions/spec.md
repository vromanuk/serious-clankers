# Skeptic Conventions

## Intent

Stage 9 of skeptic: judge plain words in code, one function per task, and clear Rust style — not comments, naming, architecture, or hard bans.

## Triggers

- **SHOULD** apply when skeptic runs stage 9, or the user asks only for composition or Rust style review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Composition pass

When multi-step functions are in scope, the agent SHALL report `composition: ok` or composition findings for monoliths and one-use rename helpers. The agent SHALL NOT ask for a split on length alone; a split must leave pieces that each read on their own.

#### Scenario: Monolith handler

- **GIVEN** load + validate + write + notify inline with no named steps
- **WHEN** reviewing conventions
- **THEN** the agent flags composition with path:line evidence

#### Scenario: Long function that reads top to bottom

- **GIVEN** a long function with a simple signature that does one job in clear steps
- **WHEN** reviewing conventions
- **THEN** the agent does not ask for a split because of length

### Behavior: Easy to read

The agent SHALL flag clever but hard-to-follow code when a simpler shape would read better, unless a real measured hot path has a short local why.

#### Scenario: Clever one-liner

- **GIVEN** a dense one-liner that obscures a simple branch
- **WHEN** reviewing conventions
- **THEN** the agent flags it and prefers a plainer structure unless the hot path is documented

### Behavior: Assertions for bugs

The agent SHALL flag an invariant the code relies on that has neither an assertion nor a type enforcing it, and compound assertions that hide which condition failed. Panics on outside input belong to complexity (error design).

#### Scenario: Constants that must agree, unchecked

- **GIVEN** `TRANSFERS_PER_BATCH_MAX = 8_000`, `TRANSFER_SIZE_BYTES = 32`, and `MESSAGE_BODY_SIZE_MAX_BYTES = 1 << 20` in one module, with nothing tying them together
- **WHEN** reviewing conventions
- **THEN** the agent suggests `const _: () = assert!(TRANSFERS_PER_BATCH_MAX * TRANSFER_SIZE_BYTES <= MESSAGE_BODY_SIZE_MAX_BYTES);` so a change to one constant that breaks the relation fails to compile

### Behavior: Settings written out

The agent SHALL flag clients and consumers that rely on library defaults for settings that decide correctness or failure behavior.

#### Scenario: Kafka consumer on default commit settings

- **GIVEN** a Kafka consumer config that sets only `bootstrap.servers` and `group.id`, in code that processes each message before it should be committed
- **WHEN** reviewing conventions
- **THEN** the agent flags that `enable.auto.commit` defaults to true and `isolation.level` defaults differ between clients, and suggests setting `enable.auto.commit`, `auto.offset.reset`, and `isolation.level` explicitly

### Behavior: One copy of each fact

The agent SHALL flag stored values that duplicate derivable ones and checks whose results are used after the checked state may have changed.

#### Scenario: Count stored beside its collection

- **GIVEN** a struct with `free_slots: Vec<usize>` and `free_count: usize`, updated in separate places
- **WHEN** reviewing conventions
- **THEN** the agent flags that the two can disagree and suggests deriving the count from `free_slots.len()`

#### Scenario: Check used after the state changed

- **GIVEN** code that sets `has_free_slot = !free_slots.is_empty()`, calls a method that can take slots, then pops with `unwrap()` when `has_free_slot` is true
- **WHEN** reviewing conventions
- **THEN** the agent flags the stale check and suggests `let Some(index) = free_slots.pop() else { … }` at the point of use

## Constraints

### Constraint: Not a linter

The agent MUST NOT use this stage to re-litigate `rustfmt` / `clippy` output.

### Constraint: Stage boundary

The agent MUST NOT expand this stage into full comment or naming review, and MUST NOT judge whether composition leaves are **public** (architecture `### Component API`).

<!-- skillet-version: 1.8.0 -->
