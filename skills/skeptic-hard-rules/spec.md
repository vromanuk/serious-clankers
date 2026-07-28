# Skeptic Hard Rules

## Intent

Stage 8 of skeptic: pass/fail absolute bans only, from `references/hard-rules.md` on this skill. No soft taste.

## Triggers

- **SHOULD** apply when skeptic runs stage 8, or the user asks only for hard-rules review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Checklist only

The agent SHALL walk `references/hard-rules.md` against the diff and report only rule-id hits with locators, or `none`.

#### Scenario: Private import

- **GIVEN** a diff imports another component’s private path
- **WHEN** reviewing hard rules
- **THEN** the agent reports `HR-private-import` with path:line and smallest fix

### Behavior: No invented IDs

The agent SHALL NOT invent hard-rule IDs not present in that hard-rules file.

#### Scenario: Soft nit

- **GIVEN** only a naming preference with no rule id
- **WHEN** reviewing hard rules
- **THEN** the agent leaves it out of this section (or routes to comments/naming/conventions), not a fake HR id

## Constraints

### Constraint: Pass/fail only

The agent MUST NOT expand this stage into architecture or pure-core essays.

<!-- skillet-version: 1.7.0 -->
