# PR Description

## Intent

Make the agent write pull-request descriptions that always use the same labeled structure a reviewer can scan: **Why** (real need), **How** (mechanism), then **Testing** when applicable. This exists because unguided agents invent freeform layouts, dump file lists, bury the need, and put empty testing placeholders.

## Triggers

- **SHOULD** apply when the user asks to write, draft, or generate a PR description, fill in a PR body, or asks what to put in the pull request.
- **SHOULD** apply when the user asks to summarize the branch or diff *for a PR* (not a free-form essay).
- **SHOULD NOT** apply when the user only wants a code review (use skeptic) without a PR write-up.
- **SHOULD NOT** apply when the user only wants a git commit message (use a commit skill if present).
- **SHOULD NOT** apply when the user wants the feature implemented rather than described.

## Behaviors

### Behavior: Fixed label structure

The agent SHALL always emit the PR body with these exact line-start labels in this order, each on its own line as `Label: ` followed by content (or a short block starting on the next line):

1. `Why:` — real need / problem in plain words
2. `How:` — what changed and how it works
3. `Testing:` — what was run or what the reviewer should run (required unless the user explicitly asked for Why/How only)

Optional after those, still as a labeled line: `Risks:` or `Notes:` when material. The agent MUST NOT use freeform `## Summary` / `## How` markdown section titles as a substitute for these labels. The agent MUST NOT reorder or rename the required labels.

#### Scenario: Config whitelist fix

- **GIVEN** a branch that adds two gateway IDs to a config list
- **WHEN** the user asks for a PR description
- **THEN** the body starts with `Why:` stating the sites were not whitelisted, then `How:` stating the IDs were added to the named config list, using those exact labels

### Behavior: Why before how

The agent SHALL put the real need under `Why:` and the mechanism under `How:`. `Why:` MUST come first. Content under `How:` MUST NOT appear before `Why:`.

#### Scenario: Feature branch PR

- **GIVEN** a branch that adds input validation to a public API
- **WHEN** the user asks for a PR description
- **THEN** `Why:` states the need and high-level idea before `How:` gives implementation detail

### Behavior: Component diagram when structure matters

When the change crosses multiple components or changes boundaries, the agent SHALL include a plain-text diagram and/or changed-components sketch. That block MUST sit in its own section between horizontal rule lines (`---`) so it is separated from `Why:`, `How:`, and `Testing:` — not inline inside the `How:` paragraph. Order: `Why:` → `How:` → `---` → diagram/components → `---` → `Testing:`. When the change is a tiny single-site fix, the agent MAY omit the diagram block entirely (no empty `---` pair).

#### Scenario: Multi-module change

- **GIVEN** a branch that introduces a new pure planner called from a shell handler
- **WHEN** the user asks for a PR description
- **THEN** the description includes a short diagram or structured sketch of how the pieces connect, placed between `---` lines after `How:` and before `Testing:`

### Behavior: Testing section substance

The agent SHALL fill `Testing:` with what was actually run or what the reviewer should run (commands, cases, manual checks). The agent MUST NOT use empty placeholders like “N/A”, “tested”, or “no testing required” without saying what was checked or why automation is hard and what the manual check is.

#### Scenario: Tests already run

- **GIVEN** the branch added a unit test and `cargo test` was the natural check
- **WHEN** writing the PR description
- **THEN** `Testing:` names the relevant tests or command and what they cover, not a bare “passed”

### Behavior: Diff-grounded accuracy

The agent SHALL inspect **committed** branch changes before drafting: find the merge base against the default base branch (or the base the user named), list files changed in those commits, and read the committed diff enough to understand the change. The description MUST be grounded in that evidence (plus issue text the user gave). The agent MUST NOT invent features, files, or test runs that are not in that evidence. The agent MUST NOT rely only on uncommitted working-tree edits when the PR is about committed branch work; mention dirty uncommitted files only if the user asked to include WIP.

#### Scenario: Only validation fix in the diff

- **GIVEN** the committed branch diff only tightens null checks with no new endpoints
- **WHEN** the user asks for a PR description
- **THEN** the description describes that validation fix and does not claim unrelated features or new services

#### Scenario: Agent must look at commits first

- **GIVEN** a feature branch with several commits ahead of main
- **WHEN** the user asks for a PR description
- **THEN** the agent lists or otherwise uses the files changed by those commits and drafts from that committed set, not from memory or an empty assumption

## Constraints

### Constraint: No corporate fluff

The agent MUST NOT write padded HR/corporate language, vague slogans, or long file inventories that replace the why/how narrative.

### Constraint: No invented validation

The agent MUST NOT claim tests, deploys, or manual checks that were not evidenced in the session or repo unless the user explicitly stated them.

### Constraint: Do not implement as the PR description task

The agent MUST NOT treat a PR-description request as permission to implement new product features; description is the deliverable unless the user also asks to code.

<!-- skillet-version: 1.7.0 -->
