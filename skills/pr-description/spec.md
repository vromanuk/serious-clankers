# PR Description

## Intent

Make the agent write pull-request descriptions with the same labeled structure every time, so a reviewer can scan them: **Why** (the missing contract and its consequence), **How** (intuition for the approach, then the mechanism), **Testing** (what was checked). Unguided agents invent freeform layouts, dump file lists, bury the need, narrate incidents instead of explaining the defect, and leave empty testing placeholders.

## Triggers

- **SHOULD** apply when the user asks to write, draft, or generate a PR description, fill in a PR body, or asks what to put in the pull request.
- **SHOULD** apply when the user asks to summarize the branch or diff *for a PR* (not a free-form essay).
- **SHOULD NOT** apply when the user only wants a code review (use skeptic) without a PR write-up.
- **SHOULD NOT** apply when the user only wants a git commit message (use a commit skill if present).
- **SHOULD NOT** apply when the user wants the feature implemented rather than described.

## Behaviors

### Behavior: Fixed label structure

The agent SHALL emit the PR body with these line-start labels, in this order, each as `Label: ` followed by content: `Why:`, `How:`, `Testing:`. `Testing:` is required unless the user asked for Why/How only. An optional `Notes:` (review tips, risks, follow-ups) MAY follow `Testing:`. The agent MUST NOT rename or reorder the labels, substitute freeform markdown headings (`## Summary`), or add an `Idea:` label.

#### Scenario: Config whitelist fix

- **GIVEN** a branch that adds two gateway IDs to a config list
- **WHEN** the user asks for a PR description
- **THEN** the body starts with `Why:` stating the sites were not whitelisted, then `How:` stating the IDs were added to the named config list

### Behavior: Why before how (need + intuition first)

Under `Why:` the agent SHALL state the missing contract and its consequence in the system's own names, so a reviewer new to the area understands the need:

- For a feature: the value that is not available and the caller that cannot get it.
- For a fix: what the code does or fails to guarantee, stated as the precise technical mechanism, followed by the observed impact in a sentence or two as evidence. `Why:` SHALL open with the mechanism, not with an incident narrative or a date.
- Team shorthand SHALL be replaced with plain words or defined where first used.

`Why:` MUST NOT be a file list or describe the implementation. Under `How:` the agent SHALL give the intuition for the approach first (for example, the existing pattern it extends), then the mechanism.

#### Scenario: Feature branch PR

- **GIVEN** a branch that adds input validation to a public API
- **WHEN** the user asks for a PR description
- **THEN** `Why:` states that empty input is accepted so callers get a bad success, and `How:` gives the approach and then the implementation

#### Scenario: Reliability fix after an outage

- **GIVEN** a branch that adds a missing connect timeout after a deploy caused errors
- **WHEN** the user asks for a PR description
- **THEN** `Why:` opens with the missing timeout and what it makes the code do (a TCP connect blocking until the kernel stops retrying the SYN), then gives the outage briefly as evidence; it does not open with "In the <date> rollout"

#### Scenario: Why is only a file dump

- **GIVEN** a multi-file feature branch
- **WHEN** the user asks for a PR description
- **THEN** `Why:` states the problem, not the changed paths

### Behavior: Enabling refactor explained

When the diff includes a refactor that exists only to make the change possible, the agent SHALL say under `How:` why it was needed and whether it changes behavior. The agent MAY say under `Notes:` how to review it.

#### Scenario: Constructor added so a new config field does not touch every caller

- **GIVEN** a branch that adds a config field, plus a constructor and a migration of hand-written configs to it
- **WHEN** the user asks for a PR description
- **THEN** `How:` explains that the field alone would have required editing every hand-written config, that the constructor removes that cost, and that the migration changes no behavior

### Behavior: How in full sentences, decisions only

Under `How:` the agent SHALL open with a full sentence that names what the PR adds or changes, and SHALL write every bullet as a full sentence a reviewer understands without the diff. The agent MUST NOT use noun-phrase openers ("PRD values for …") or fragments joined by colons or semicolons. `How:` SHALL keep only the decisions a reviewer should judge; it MUST NOT repeat what code comments, values files or the ticket already carry, or list side changes (image bumps, doc or link tweaks). When the change is one step of a rollout or migration, the agent SHALL state the plan as its own bullet, including when the new component starts being used, and MUST NOT describe it as unused if a later step uses it.

#### Scenario: Deploy step of a migration

- **GIVEN** a config-only branch that deploys a new relay to PRD next to the legacy one, plus an image bump and a doc update
- **WHEN** the user asks for a PR description
- **THEN** `How:` opens with "Adds helm values for … in …", has a bullet for the migration plan (who uses the new bucket first, when the rest move), states sizing as a full sentence with its reason, and does not list the image bump or the doc update

### Behavior: Component diagram when structure matters

When the change crosses components or changes boundaries, the agent SHALL include a plain-text diagram or changed-components sketch in its own block between two `---` lines, after `How:` and before `Testing:`, never inside the `How:` paragraph. For a single-site change the agent SHALL omit the block (no empty `---` pair).

#### Scenario: Multi-module change

- **GIVEN** a branch that introduces a new pure planner called from a shell handler
- **WHEN** the user asks for a PR description
- **THEN** a short sketch of how the pieces connect sits between `---` lines after `How:` and before `Testing:`

### Behavior: Testing section substance

The agent SHALL keep `Testing:` short: which tests cover the change and where it was verified, with a link to the ticket or PR that holds the evidence. It MUST NOT include command output, counts or step lists. For a config-only or deploy PR, the agent SHALL point to the tests and verification of the code being deployed. The agent MUST NOT write a bare "N/A", "tested", or "no testing required".

#### Scenario: Tests already run

- **GIVEN** the branch added a unit test and `cargo test` was the natural check
- **WHEN** writing the PR description
- **THEN** `Testing:` names the test or command and what it covers, not a bare "passed"

#### Scenario: Config-only deploy PR

- **GIVEN** a branch that only adds helm values for code merged earlier and verified in eng
- **WHEN** writing the PR description
- **THEN** `Testing:` is one sentence: covered by unit tests and verified in eng, with the ticket link; no helm output or check lists

### Behavior: Diff-grounded accuracy

The agent SHALL read the **committed** branch changes before drafting: find the merge base against the base branch the user named (else the default branch), list the files changed by those commits, and read enough of the diff to explain the change. The description MUST be grounded in that evidence plus any ticket or issue text. Uncommitted edits are excluded unless the user asked to include work in progress.

#### Scenario: Only validation fix in the diff

- **GIVEN** the committed branch diff only tightens null checks with no new endpoints
- **WHEN** the user asks for a PR description
- **THEN** the description describes that validation fix and claims no unrelated features or services

#### Scenario: Agent must look at commits first

- **GIVEN** a feature branch with several commits ahead of main
- **WHEN** the user asks for a PR description
- **THEN** the agent drafts from the files changed by those commits, not from memory or commit subjects alone

## Constraints

### Constraint: Only what a reviewer needs

The agent MUST NOT write corporate filler, "please review" checklists, or detail that does not help a reviewer judge the change: file or env-var inventories, commit-by-commit lists, raw metric dumps, reproduction commands, line counts. When a ticket or commit already holds such detail, the agent points to it.

### Constraint: No invented validation

The agent MUST NOT claim features, files, tests, deploys, or manual checks that the diff, the session, or the user do not support.

### Constraint: Do not implement as the PR description task

The agent MUST NOT treat a PR-description request as permission to implement product changes; the description is the deliverable unless the user also asks to code.

<!-- skillet-version: 1.7.0 -->
