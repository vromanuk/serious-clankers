# Skeptic

## Intent

Skeptic is a standalone read-only multi-stage code review, Rust-first. It snapshots scope and real need, always runs nine fixed stages (purpose → architecture → testability → unit-tests → observability → comments → naming → conventions → hard-rules) in the current session without spawning subagents, coordinates findings, and hands off a single report. It does not implement fixes unless asked and is not an implement→fix→verify loop.

## Triggers

- **SHOULD** apply when the user asks for review, code review, comprehensive review, review of a branch/PR/diff, or invokes skeptic.
- **SHOULD NOT** apply for post-implement fix/verify loops (that is a different job).
- **SHOULD NOT** apply for file-by-file progressive readability campaigns.
- **SHOULD NOT** apply for skill authoring, brainstorming, or pure CI babysitting alone.

## Behaviors

### Behavior: Fixed nine-stage pipeline

The agent SHALL always run stages purpose, architecture, testability, unit-tests, observability, comments, naming, conventions, and hard-rules in that report order, and SHALL NOT replace them with a freestyle product essay. The agent MUST NOT skip unit-tests or treat it as optional depth under testability.

#### Scenario: User asks to review a PR

- **GIVEN** a PR scope and an explicit review request
- **WHEN** skeptic runs
- **THEN** the handoff contains all nine sections in order (findings or N/A with reason each), including `## 4. Unit tests`

### Behavior: Same-session, no spawned agents

The agent SHALL run stages 1→9 in the current session without spawning subagents, still loading each stage contract, and SHALL NOT freestyle a one-lens product essay.

#### Scenario: Review requested

- **GIVEN** a review request and a harness that can spawn subagents
- **WHEN** skeptic runs
- **THEN** the agent completes all nine stages in this session and does not spawn subagents

### Behavior: Evidence-backed coordination

The agent SHALL accept only evidence-backed, stage-appropriate findings, reject preference-only or evidence-free concerns, and merge into one report using the skeptic concern format.

#### Scenario: Mixed weak and strong findings

- **GIVEN** stage reports include one vague taste nit and one path:line contract bug
- **WHEN** coordinating
- **THEN** the agent drops or demotes the taste nit and keeps the contract bug with severity and evidence labels

### Behavior: Read-only unless asked

The agent SHALL NOT implement fixes unless the user explicitly asked to fix.

#### Scenario: Review-only request

- **GIVEN** the user asked only to review
- **WHEN** material findings exist
- **THEN** the agent reports recommended next steps and asks before changing code

## Constraints

### Constraint: No invented hard-rule IDs

The agent MUST NOT invent hard-rule IDs; stage 9 uses only IDs from `skeptic-hard-rules/references/hard-rules.md`.

<!-- skillet-version: 1.7.0 -->
