# Skeptic SOLID

## Intent

Review whether types, traits, modules, and crates are shaped so the next likely change is small and local: the five SOLID class principles applied to Rust, Noback's package principles (cohesion and coupling, with instability and abstractness metrics) applied to crates and modules, and Dan North's CUPID properties as a counterweight against over-applying them. Runs standalone, or as an extra `## SOLID` section of a skeptic review when the diff touches design structure.

## Triggers

- **SHOULD** apply when the user asks for a SOLID, design-principles, or package-design review, or when skeptic runs and the diff adds or changes traits, trait implementations, types with behavior, module layout, crate boundaries, `Cargo.toml` dependencies, or a library crate's public API.
- **SHOULD NOT** apply to diffs with none of these.

## Behaviors

### Behavior: Change pressure required

The agent SHALL name the concrete change pressure behind every finding — the group of people asking for changes, the variant that keeps being added, the confused caller, or the volatile dependency — and SHALL NOT report a principle name alone.

#### Scenario: Split justified only by the acronym

- **GIVEN** a small struct whose content and formatting always change together
- **WHEN** reviewing SOLID
- **THEN** the agent does not ask to split it, and notes that splitting would create artificial seams

### Behavior: Single responsibility

The agent SHALL flag types or modules that answer to different groups of people, using Noback's symptoms (many fields, many public methods, methods using disjoint fields, specific tasks in private helpers) as evidence.

#### Scenario: Building and sending in one type

- **GIVEN** a mailer struct that translates the subject, renders the body, and sends through SMTP, so every test needs all three collaborators
- **WHEN** reviewing SOLID
- **THEN** the agent flags two reasons to change and suggests a separate unit that builds the message, leaving the mailer to send it

### Behavior: Open/closed without speculative extension points

The agent SHALL flag the same kind-switch repeated across functions when the set keeps growing from outside, and SHALL keep closed sets the crate owns as enums.

#### Scenario: Format matched in three functions

- **GIVEN** a `match format` on strings repeated in encode, content-type, and file-extension functions, with new formats requested by users
- **WHEN** reviewing SOLID
- **THEN** the agent flags the repeated switch and offers a trait with implementations registered at the app edge

#### Scenario: Closed enum matched in one place

- **GIVEN** an enum of three message kinds owned by the crate and matched in one function
- **WHEN** reviewing SOLID
- **THEN** the agent does not suggest a trait

### Behavior: Substitutable implementations

The agent SHALL flag trait implementations that panic, return unsupported, silently do nothing, reject inputs the trait accepts, or force callers to check which implementation they have, and SHALL prefer adding the method, splitting the trait, or removing the abstraction.

#### Scenario: unimplemented! in a trait impl

- **GIVEN** a `File` trait with `change_owner`, and a `DropboxFile` implementation that calls `unimplemented!()`
- **WHEN** reviewing SOLID
- **THEN** the agent flags broken substitution and suggests a separate `FileWithOwner: File` trait required only where ownership is changed

### Behavior: When a trait is worth adding

The agent SHALL flag traits with one production implementation whose only other implementation is a test mock, `FooService` + `FooServiceImpl` pairs, and traits declared on the implementing side of a boundary; and SHALL accept traits for I/O, isolated third-party code, open sets, and replaceable parts of a composed graph.

#### Scenario: Trait only for a mock

- **GIVEN** a new `trait PriceCalculator` with one implementation and a mock used in tests, for a pure calculation
- **WHEN** reviewing SOLID
- **THEN** the agent flags the unnecessary abstraction and suggests a concrete struct tested through its real API

#### Scenario: Domain depends on the database driver

- **GIVEN** an authentication policy that holds a connection pool and runs SQL itself
- **WHEN** reviewing SOLID
- **THEN** the agent flags policy depending on mechanism and suggests a role trait owned by the policy (for example `UserProvider`), implemented in an infrastructure module

### Behavior: Package cohesion and coupling

When crates or their dependencies change, the agent SHALL check release/reuse, common reuse, common closure, acyclic dependencies (including module cycles and dev-dependencies), stable dependencies, and stable abstractions, and MAY report instability, abstractness, and distance from the main sequence as hints.

#### Scenario: Core crate gains a framework dependency

- **GIVEN** a domain-types crate used by most of the workspace that adds a dependency on a web framework
- **WHEN** reviewing SOLID
- **THEN** the agent flags a stable crate depending on a volatile one and suggests keeping framework code in a leaf crate that depends on the core

#### Scenario: Application split into a crate per layer

- **GIVEN** a single application split into separate crates for each layer of each subdomain
- **WHEN** reviewing SOLID
- **THEN** the agent suggests modules instead, since nothing reuses those crates independently

## Constraints

### Constraint: Stage boundary

The agent MUST NOT repeat component layout, public surface, data ownership, or dependency passing (architecture), depth and conjoined splits (complexity), names (naming), Rust style (conventions), or telemetry (observability); it cross-references or drops those findings instead. The agent MUST NOT gate on metrics.

<!-- skillet-version: 1.7.0 -->
