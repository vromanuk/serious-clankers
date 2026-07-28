# Skeptic Comments

## Intent

Stage 5 of skeptic: judge comment quality only — needed?, clear English, why not what, public docs vs body comments, design headers. Not naming, composition, or hard bans.

## Triggers

- **SHOULD** apply when skeptic runs stage 5, or the user asks only for comment review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Why not what; necessary comments

The agent SHALL flag restating and unnecessary comments, prefer *why* or clearer code, and SHALL apply the comment review checklist (clear English; simplify unclear code; short *what* OK for regex/hard algorithms; keep decision/scar info).

#### Scenario: Obvious restatement

- **GIVEN** a comment such as “find the element in the vector” above a clear find/contains check
- **WHEN** reviewing comments
- **THEN** the agent flags it as stating the obvious

### Behavior: Design headers

The agent SHALL flag design headers that only restate types and SHALL prefer purpose + given/expected without type lines.

#### Scenario: Type-line design header

- **GIVEN** a comment that is only a type signature above a function
- **WHEN** reviewing comments
- **THEN** the agent flags it

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into naming, composition, architecture, or a full hard-rule walk.

<!-- skillet-version: 1.7.0 -->
