# Skeptic Comments

## Intent

Stage 7 of skeptic: judge comment quality only — needed?, clear English, why/benefit not bare what or unexplained rules, module `//!` shape, public docs vs body comments, design headers. Not naming, composition, or hard bans.

## Triggers

- **SHOULD** apply when skeptic runs stage 7, or the user asks only for comment review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Why not what; necessary comments

The agent SHALL flag restating and unnecessary comments, prefer *why* / benefit / rejected alternative or clearer code, and SHALL apply the comment review checklist (clear English; simplify unclear code; short *what* OK for regex/hard algorithms; keep decision/scar info). The agent SHALL flag module or API docs that state design rules without explaining why when that rule is a non-obvious choice. The agent SHALL flag public docs that describe internal mechanisms callers do not need.

#### Scenario: Obvious restatement

- **GIVEN** a comment such as “find the element in the vector” above a clear find/contains check
- **WHEN** reviewing comments
- **THEN** the agent flags it as stating the obvious

#### Scenario: Rule without why

- **GIVEN** a module `//!` that only states “clients query views, not raw keys” (or similar) with no benefit or alternative explained
- **WHEN** reviewing comments
- **THEN** the agent flags it and prefers intuition + why + rules that follow (`comments.md` template)

#### Scenario: Implementation detail in public docs

- **GIVEN** a `///` on a public `is_ready()` that explains its internal state machine instead of what "ready" means to callers
- **WHEN** reviewing comments
- **THEN** the agent flags it and asks for caller-visible meaning in the doc, with the internals moved into the body

#### Scenario: Public fallible function without an errors section

- **GIVEN** a new `pub fn write_snapshot(...) -> Result<(), SnapshotError>` whose doc says only what it writes, while `SnapshotError` has a retryable `Timeout` and a non-retryable `Rejected`
- **WHEN** reviewing comments
- **THEN** the agent asks for a `# Errors` section that lists both variants, when each happens, and whether retrying helps

### Behavior: Precise wording

The agent SHALL flag comments that paraphrase where a standard technical term exists, that use a vague quality instead of naming the behavior, that document a safety type without the threat it stops, or that state behavior of other code without it having been checked, and SHALL suggest wording per `comments.md` § Precise wording.

#### Scenario: Paraphrase instead of the term

- **GIVEN** a module doc “Whether a Client may connect, and what each of its Queries may do.” on code that verifies tokens and checks statements
- **WHEN** reviewing comments
- **THEN** the agent flags it and suggests naming authentication and authorization and how each is applied

#### Scenario: A person's verb for a system

- **GIVEN** a README line “the service answers questions about your orders” or a test comment “the server answers the next query”
- **WHEN** reviewing comments and docs in the diff
- **THEN** the agent flags it and suggests the operation's verb (“returns order records matching a filter”, “the server returns rows from the newer file”)

#### Scenario: Vague quality

- **GIVEN** a type documented as “the locked-down database”
- **WHEN** reviewing comments
- **THEN** the agent flags “locked-down” and asks which settings make it so and what they refuse

### Behavior: Settings and numbers explained

The agent SHALL check docs on settings (config fields, builder methods, public constants) for what the setting controls, what happens at the limit, special values, relations to other limits, and the default, and SHALL flag setting docs that tell the incident behind the setting instead of its general behavior. The agent SHALL flag chosen numbers (defaults, constants, literals) whose value has no stated reason or only a provenance note.

#### Scenario: Timeout doc tells the incident

- **GIVEN** a `connect_timeout` field doc that describes a server restart, replicas, and retrying elsewhere, but not what happens when the timeout is reached
- **WHEN** reviewing comments
- **THEN** the agent flags it and suggests: what the timeout limits, what fails at the limit, and a link to the default

#### Scenario: Default copied from elsewhere

- **GIVEN** `default_keep_alive_timeout() -> 10 s` documented only as "copied from the previous service's configuration"
- **WHEN** reviewing comments
- **THEN** the agent flags that the value has no reason and asks what the 10 s is chosen against

### Behavior: Docs of renamed or changed items

The agent SHALL re-read the doc comment of every function or type the diff renames or whose behavior it changes, even when the diff does not touch that comment, and SHALL flag a doc that no longer describes what the item does.

#### Scenario: Renamed function keeps a doc about something else

- **GIVEN** a diff that renames `find_distinct_columns` to `check_unique_columns`, and the unchanged doc above it describes the time column instead of the duplicate check the function does
- **WHEN** reviewing comments
- **THEN** the agent flags the doc as stale and suggests one that states the result and each way the function fails

### Behavior: Design headers

The agent SHALL flag design headers that only restate types or only restate the function name and SHALL prefer purpose (with why when the shape is a choice) + given/expected without type lines.

#### Scenario: Type-line design header

- **GIVEN** a comment that is only a type signature above a function
- **WHEN** reviewing comments
- **THEN** the agent flags it

## Constraints

### Constraint: Stage boundary

The agent MUST NOT expand this stage into naming, composition, architecture, or a full hard-rule walk.

<!-- skillet-version: 1.7.0 -->
