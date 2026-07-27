# Explain Diff

## Intent

Explain a code change (diff, branch, or PR) so the reader builds a solid mental model: first principles and intuition before detail, with real alternatives considered. Not a line dump, not a quiz, not a multi-lens review (that is skeptic).

## Triggers

- **SHOULD** apply when the user asks to explain a diff, branch, PR, or code change with teaching/understanding as the goal (rich walkthrough, intuition-first).
- **SHOULD** apply when the user wants a beginner-friendly story of “what changed and why” grounded in the real diff.
- **SHOULD NOT** apply when the user only wants a PR body (use pr-description).
- **SHOULD NOT** apply when the user wants multi-lens code review (use skeptic).
- **SHOULD NOT** apply when the user wants a generic concept lecture with no change in scope (use explain-topic).
- **SHOULD NOT** apply when the primary goal is implementing or fixing code.

## Behaviors

### Behavior: Grounded in the real change

The agent SHALL inspect the actual branch/diff (and surrounding code needed for background) and SHALL NOT invent files, behaviors, or tests absent from evidence.

#### Scenario: Small validation fix

- **GIVEN** the committed diff only tightens null checks
- **WHEN** the user asks for an explain-diff write-up
- **THEN** the explanation covers that validation fix and does not invent unrelated systems

### Behavior: Background deep then narrow

The agent SHALL give deep background for beginners (marked skippable if already known) and then narrow background for the code paths this change touches, after exploring surrounding code.

#### Scenario: Change in a multi-module system

- **GIVEN** a change to a pure planner used by a shell handler
- **WHEN** explaining the diff
- **THEN** deep background orients the reader to the surrounding system (skippable), then narrows to the planner/handler paths that matter

### Behavior: Intuition and first principles before detail

The agent SHALL lead with first principles and core intuition (essence, analogy when helpful, toy data, diagrams) before a high-level code walkthrough. The agent SHALL put mechanism detail after the mental model is in place.

#### Scenario: User asks what this PR does

- **GIVEN** a non-trivial PR
- **WHEN** the agent explains
- **THEN** the write-up starts with the real need and a simple picture of the idea, not a file-by-file list

### Behavior: Alternatives considered

The agent SHALL present real alternative approaches (at least when the design is non-obvious), with plain pros/cons and why this change chose its path (or label assumed if only inferred from the diff).

#### Scenario: Material design fork

- **GIVEN** the change could have been done with a different structure
- **WHEN** explaining
- **THEN** the write-up names 2–3 real options and why the chosen one fits

### Behavior: Code walkthrough after model

The agent SHALL walk through the code changes at a high level, grouped in an understandable order, after intuition and alternatives. The agent MUST NOT open with a raw file dump.

#### Scenario: Multi-file change

- **GIVEN** several files changed
- **WHEN** explaining
- **THEN** changes are grouped by idea or flow, not only by path alphabetically

### Behavior: Output format

The agent SHALL default to a clear markdown explanation (ASCII or simple HTML/CSS diagrams allowed). When the user asks for a rich HTML page, the agent SHALL write one self-contained HTML file outside the repo with a `YYYY-MM-DD-` filename prefix (e.g. under `/tmp` or a path the user named). The agent MUST NOT include an interactive quiz or flashcard section.

#### Scenario: HTML request

- **GIVEN** the user asks for an HTML explanation of the branch
- **WHEN** the agent finishes
- **THEN** a single self-contained HTML file exists at a path outside the code repo whose name starts with today’s date

## Constraints

### Constraint: No quiz

The agent MUST NOT produce a quiz, flashcards, or multi-choice test section for this skill.

### Constraint: Teaching not shipping

The agent MUST NOT treat explain-diff as permission to implement product features; the explanation is the deliverable unless the user also asks to code.

### Constraint: Plain language

The agent MUST use plain words, lead with intuition, and separate facts from assumptions. The agent MUST NOT use corporate filler or bury the real need under jargon.

<!-- skillet-version: 1.7.0 -->
