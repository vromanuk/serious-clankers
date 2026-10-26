# Skeptic

## Intent

Skeptic is a standalone read-only multi-stage code review, Rust-first. It snapshots scope and real need, always runs ten fixed stages (purpose → architecture → complexity → testability → unit-tests → observability → comments → naming → conventions → hard-rules) in the current session without spawning subagents, coordinates findings, and hands off a single report. It does not implement fixes unless asked and is not an implement→fix→verify loop.

## Triggers

- **SHOULD** apply when the user asks for review, code review, comprehensive review, review of a branch/PR/diff, or invokes skeptic.
- **SHOULD NOT** apply for post-implement fix/verify loops (that is a different job).
- **SHOULD NOT** apply for file-by-file progressive readability campaigns.
- **SHOULD NOT** apply for skill authoring, brainstorming, or pure CI babysitting alone.

## Behaviors

### Behavior: Fixed ten-stage pipeline

The agent SHALL always run stages purpose, architecture, complexity, testability, unit-tests, observability, comments, naming, conventions, and hard-rules in that report order, and SHALL NOT replace them with a freestyle product essay. The agent MUST NOT skip unit-tests or treat it as optional depth under testability.

#### Scenario: User asks to review a PR

- **GIVEN** a PR scope and an explicit review request
- **WHEN** skeptic runs
- **THEN** the handoff contains all ten sections in order (findings or N/A with reason each), including `## 3. Complexity` and `## 5. Unit tests`

### Behavior: Fixed file scope before stage 1

Before stage 1 the agent SHALL list the files it will review — the production files that carry the change, then their tests, with only the diff of shared files the change touches — and SHALL leave out build and generated files (BUCK, lockfiles, generated code, deploy templates) unless the change is about them. The agent SHALL state that list to the user and SHALL read every scoped production file in full before stage 1.

#### Scenario: Branch touches code, tests and build files

- **GIVEN** a branch that changes Rust modules, their tests, a BUCK file and a lockfile
- **WHEN** skeptic starts
- **THEN** the agent states a scope listing the Rust modules and tests, notes that BUCK and the lockfile are left out, and the report's `Scope:` line repeats it

### Behavior: One stage per pass

The agent SHALL run each stage as its own pass over the whole file scope: load that stage's contract and the references it names, walk every scoped file with only that stage's question, and record that stage's findings (or `ok` / `none` with a reason) before starting the next stage. The agent SHALL NOT apply several stages in one pass, and SHALL NOT judge a stage from memory of an earlier pass without re-opening the code it cites.

#### Scenario: A naming problem is noticed during the architecture stage

- **GIVEN** stage 2 is running and the agent notices a function named as a noun
- **WHEN** it records stage 2 findings
- **THEN** the naming concern is not reported under architecture; stage 8 re-walks the scoped files and reports it there with path:line

### Behavior: Visible progress per stage

The agent SHALL keep one task per step — scope, each of stages 1–10, `## Async` and `## SOLID` when they apply, and final checks — and SHALL mark each done as it finishes.

#### Scenario: Long review in progress

- **GIVEN** a review of a dozen files
- **WHEN** stage 6 is running
- **THEN** the task list shows scope and stages 1–5 done, stage 6 in progress, and the rest pending

### Behavior: Reliability and security inside architecture

Stage 2 (architecture) SHALL include `### Reliability` and `### Security` subsections, following the `skeptic-reliability` and `skeptic-security` contracts, when the diff touches failure-relevant code or a trust boundary; otherwise each SHALL be one line with `none` and the reason. This adds no numbered stage.

#### Scenario: Diff adds a remote call with retries

- **GIVEN** a diff that adds an object-storage upload with a retry loop and a new service-account token
- **WHEN** skeptic runs
- **THEN** `## 2. Architecture` contains `### Reliability` and `### Security`, and the report still has exactly ten numbered sections

### Behavior: Async section when the diff has async code

When the diff contains async Rust, the agent SHALL also run the `skeptic-async` contract after stage 10 and report it as `## Async`; otherwise it SHALL omit that section.

#### Scenario: Diff adds a tokio task

- **GIVEN** a diff that adds `tokio::spawn` and a `select!` loop
- **WHEN** skeptic runs
- **THEN** the handoff contains the ten numbered sections followed by `## Async`

### Behavior: SOLID section when the diff changes design structure

When the diff adds or changes traits, trait implementations, types with behavior, module layout, crate boundaries, `Cargo.toml` dependencies, or a library crate's public API, the agent SHALL also run the `skeptic-solid` contract after stage 10 (after `## Async` when present) and report it as `## SOLID`; otherwise it SHALL omit that section.

#### Scenario: Diff adds a trait and two implementations

- **GIVEN** a diff that adds a public trait with two implementations and a new workspace crate
- **WHEN** skeptic runs
- **THEN** the handoff contains the ten numbered sections followed by `## SOLID`

### Behavior: Same-session, no spawned agents

The agent SHALL run stages 1→10 in the current session without spawning subagents, still loading each stage contract, and SHALL NOT freestyle a one-lens product essay.

#### Scenario: Review requested

- **GIVEN** a review request and a harness that can spawn subagents
- **WHEN** skeptic runs
- **THEN** the agent completes all ten stages in this session and does not spawn subagents

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

### Behavior: Fix mode, stage by stage

When the user asked to fix or address issues, the agent SHALL fix each stage's intent-preserving findings at the end of that stage and run the smallest check covering them (build, clippy, touched tests) before the next stage. A finding with a real design fork SHALL be asked about or listed under `## Open decisions`, never chosen silently. When a fix adds a test for a contract, the agent SHALL break the code once to see the test fail, then restore it. After the last stage the agent SHALL run fmt, clippy and the full relevant test suites, and name any tests that cannot run and why.

#### Scenario: Review with fixes requested

- **GIVEN** the user asked for a full review and to address issues
- **WHEN** stage 4 finds an untested ordering contract and stage 2 finds a choice between two API shapes
- **THEN** stage 4 adds a test, shows it failing against a deliberately broken version and passing after restore, then checks before stage 5; the API choice appears under `## Open decisions` with lettered options

## Constraints

### Constraint: No invented hard-rule IDs

The agent MUST NOT invent hard-rule IDs; stage 10 uses only IDs from `skeptic-hard-rules/references/hard-rules.md`.

<!-- skillet-version: 1.7.0 -->
