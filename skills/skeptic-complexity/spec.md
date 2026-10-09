# Skeptic Complexity

## Intent

Stage 3 of skeptic: judge whether the change makes code harder to understand or change, using the red flags and techniques from Ousterhout's *A Philosophy of Software Design* restated in `references/complexity.md`. Each finding names the red flag, the symptom it causes, and the smallest change that removes it.

## Triggers

- **SHOULD** apply when skeptic runs stage 3, or the user asks only for a complexity or design-depth review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Red flags with evidence

The agent SHALL load `references/complexity.md` when the diff changes code, SHALL walk its review checklist on each new or changed unit, and SHALL report each finding with the red flag name, the symptom, path:line, and the smallest fix — or `complexity: ok` / `none` with a reason.

#### Scenario: Pass-through method

- **GIVEN** a new public method whose body only forwards to a field with the same signature
- **WHEN** reviewing complexity
- **THEN** the agent flags a pass-through method and offers dropping the wrapper, moving the responsibility, or merging the types

#### Scenario: Same decision in two files

- **GIVEN** an encoder and a decoder in different modules that both hard-code the same byte layout
- **WHEN** reviewing complexity
- **THEN** the agent flags information leakage and suggests one module that owns the layout

### Behavior: Depth before length

The agent SHALL treat length alone as no reason to split, and SHALL flag splits whose pieces can only be understood together.

#### Scenario: Function split for length

- **GIVEN** a function split into two methods where the second relies on state the first leaves in a field
- **WHEN** reviewing complexity
- **THEN** the agent flags conjoined methods and suggests joining them or passing the dependency through the signature

#### Scenario: Long but deep function

- **GIVEN** a 120-line function with a simple signature that reads top to bottom
- **WHEN** reviewing complexity
- **THEN** the agent does not flag its length

### Behavior: Error design in order

The agent SHALL check each new error against defining it out, masking it lower down, handling it with others in one place, or aborting, and SHALL flag errors that hide information callers need.

#### Scenario: Error callers only ignore

- **GIVEN** a delete operation that returns `NotFound` when the item is already gone, and every caller ignores it
- **WHEN** reviewing complexity
- **THEN** the agent suggests redefining the operation as "ensure it is gone" so the error disappears

#### Scenario: Swallowed error callers need

- **GIVEN** a network client that discards send failures and reports success
- **WHEN** reviewing complexity
- **THEN** the agent flags that callers cannot detect lost messages and asks for the failure to be exposed

### Behavior: Design signals routed

The agent SHALL report a hard-to-pick name or a hard-to-describe interface as a design problem, and SHALL leave the wording fix to the naming or comments stage.

#### Scenario: Name with three jobs

- **GIVEN** a new function `flush_and_maybe_commit_unless_paused`
- **WHEN** reviewing complexity
- **THEN** the agent flags that the function does several jobs and suggests splitting them, without proposing a new name

## Constraints

### Constraint: Stage boundary

The agent MUST NOT review component layout or data ownership (architecture), comment or name wording (comments, naming), or test craft (unit-tests), and MUST NOT ask for cleanup of code the change did not touch.

### Constraint: Limits are not findings

The agent MUST NOT report a red flag in a case the reference lists as a limit, such as hiding information callers need for correctness.

<!-- skillet-version: 1.7.0 -->
