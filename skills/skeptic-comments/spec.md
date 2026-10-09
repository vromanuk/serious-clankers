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
