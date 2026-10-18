# Skeptic Architecture

## Intent

Stage 2 of skeptic: judge and uphold the pack’s **default project architecture** — component-based layout (job-shaped components, public vs private surface as **use-case orchestration** not stray helpers, nested internals, data ownership, edge composition), dependency direction, and whether types force real contracts at boundaries. The same layout is the default when creating structure, not only when reviewing.

## Triggers

- **SHOULD** apply when skeptic runs stage 2, or the user asks only for architecture/component review.
- **SHOULD NOT** apply as a substitute for the full skeptic pipeline.

## Behaviors

### Behavior: Job-shaped components

The agent SHALL treat job-shaped components (public surface at the component boundary + private implementation, edge composition, data ownership) as the **default** architecture for project layout. The agent SHALL prefer that shape over layer-only layout, SHALL judge the **boundary** (what the root exports vs private modules), and SHALL flag imports past the public surface and shared write free-for-alls with path:line evidence. The agent SHALL treat the public surface as preferably **one job struct** with **use-case methods** (orchestration entrypoints) and SHALL flag shallow bags of public step-helpers, free orchestration functions that re-pass the same deps, or multi-step workflows only in handlers when a component should own them. The agent MUST NOT require folders named `api` or `internal` when the component root already defines a clear public interface (e.g. Rust `mod.rs` / `lib.rs`). The agent MUST NOT treat “no app-wide layers” as forbidding a public job struct on the component. The agent MUST NOT demand Cosmic/DDD ceremony (formal UoW/repository hierarchies) by default. When layout is in scope, the agent SHALL report `components: ok` or concrete layout findings. When the change **adds** structure, the agent SHALL judge it against this default unless the user or existing codebase clearly requires another shape. The agent SHALL expect a client that does IO to be passed in behind a narrow trait: a type parameter on the job struct when tests are the only other implementation, with the fake implemented in test code; `Arc<dyn Trait>` (with `#[async_trait]` for async methods) when configuration picks between real implementations at runtime. The agent SHALL flag test-only variants in production enums, `Arc<dyn Trait>` whose only other implementation is a test fake, and a private generic copy of the job struct that the public one only forwards to.

#### Scenario: Cross-module private import

- **GIVEN** a call into another component’s internal package
- **WHEN** reviewing architecture
- **THEN** the agent reports a finding with locator and impact

#### Scenario: Layer-only layout for a multi-job change

- **GIVEN** a change that spans multiple business jobs but only uses global controller/service/repository folders
- **WHEN** reviewing architecture
- **THEN** the agent flags missing job-shaped ownership or unclear component boundaries

#### Scenario: Stray public helpers instead of use cases

- **GIVEN** a component root that publicly exports step helpers callers must sequence (e.g. load, validate, write) for one workflow
- **WHEN** reviewing architecture
- **THEN** the agent flags the shallow surface and prefers methods on a public job struct

#### Scenario: Handler owns orchestration

- **GIVEN** an HTTP or CLI handler that performs multi-step load → domain → commit for a job that has (or should have) a component
- **WHEN** reviewing architecture
- **THEN** the agent flags missing component-level orchestration on a public job struct

#### Scenario: Public data type outside types

- **GIVEN** a `pub struct`, `pub enum`, or `pub type` that callers name, declared outside the crate’s `types.rs` or `types/` module (a root `pub use` does not count)
- **WHEN** reviewing architecture
- **THEN** the agent flags the definition site and says to add `types` if it is missing and move the type there. The job struct (`KafkaConsumer`) and a behavior trait stay with the component. A `pub(crate)` or private type stays next to its user.

#### Scenario: Free functions re-pass deps

- **GIVEN** several public free functions that each take the same store or client for one job
- **WHEN** reviewing architecture
- **THEN** the agent prefers a job struct that holds those collaborators and exposes methods

#### Scenario: Generic inner type that the public struct only forwards to

- **GIVEN** a public `AssetClient` whose only field is a private `AssetRpcCaller<R: AssetRpc>` holding the rules, where `AssetRpc` has one production implementation (the generated gRPC client) and a test fake
- **WHEN** reviewing architecture
- **THEN** the agent keeps the trait and flags the forwarding copy, offering `AssetClient<R: AssetRpc>` as the one generic job struct, so tests build the real `AssetClient` with the fake and call its public methods

#### Scenario: Test-only variant in a production enum

- **GIVEN** `enum Storage { S3 { .. }, #[cfg(test)] Fake(FakeBucket) }` inside a bucket component
- **WHEN** reviewing architecture
- **THEN** the agent flags test code in a production type and offers a narrow `ObjectStorage` trait, the bucket generic over it, and the fake implementing it in test code

#### Scenario: Implementation chosen from config at runtime

- **GIVEN** a service that runs against S3 in production and an in-memory store for local development, chosen by a config value
- **WHEN** reviewing architecture
- **THEN** the agent accepts a narrow trait held as `Arc<dyn Trait>` with `#[async_trait]`, built by one `match` at the composition root, and does not ask for a type parameter

### Behavior: Dependencies passed in

The agent SHALL flag components that build their own clients, stores, or formats, and components that reach out at runtime for config, global services, or the clock instead of receiving them through the constructor or method parameters (dependency injection). Logging and tracing globals, caches of constants, and `main` or startup code reading configuration are not findings.

#### Scenario: Config read from the environment inside a component

- **GIVEN** a component method that calls `std::env::var("BUCKET")` each time it uploads
- **WHEN** reviewing architecture
- **THEN** the agent flags the hidden dependency and suggests reading the value once at the app edge and passing it to the component's constructor

#### Scenario: Tracing global

- **GIVEN** a component that logs through the global `tracing` subscriber
- **WHEN** reviewing architecture
- **THEN** the agent does not report a hidden dependency

### Behavior: Type-driven boundaries

When APIs or shared results change, the agent SHALL check whether callers can forget a real decision, and SHALL report `type-driven: ok` or type-driven findings. When the diff adds a struct, the agent SHALL also check whether an existing struct or enum variant already carries those fields, and SHALL flag a near-copy with reuse / generalize / keep-both options.

#### Scenario: Empty list as silent success

- **GIVEN** success type is a bare list while product forbids empty
- **WHEN** reviewing architecture
- **THEN** the agent flags the boundary and prefers a type that forces the contract

#### Scenario: Caller-controlled order checked at runtime

- **GIVEN** a type whose methods must be called in a fixed order the caller controls (for example status line, then headers, then body), enforced by a stage field and a runtime error in each method
- **WHEN** reviewing architecture
- **THEN** the agent offers typestate as an option: methods only on the state that allows them, and each step takes `self` by value and returns the next state

#### Scenario: Event-driven state machine

- **GIVEN** a component whose next state is decided by incoming events (a consumer, an elevator controller, a consensus node), written as an enum of states
- **WHEN** reviewing architecture
- **THEN** the agent does not ask for typestate, because the code holding the machine cannot know its state at compile time

#### Scenario: New struct repeats an existing one

- **GIVEN** a diff adds a struct whose fields match an existing struct or enum variant under different names (for example `KafkaRecord` repeating `BrokerRecord`: topic, partition, offset, key, and payload, plus registration)
- **WHEN** reviewing architecture
- **THEN** the agent flags both types and offers reuse, generalize, or keep both. Keep-both is allowed only when the finding names the fact that would be a lie if they were one type. A comment that distinguishes the new struct from some other type is not that fact. A wrapper that holds the existing type plus the extra field is not this finding.

### Behavior: Reliability and security lenses

When the diff touches failure-relevant code, the agent SHALL follow the `skeptic-reliability` contract and report it as `### Reliability` inside this stage; when it touches a trust boundary, the agent SHALL follow the `skeptic-security` contract and report it as `### Security`. Otherwise the agent SHALL write one line per lens with `none` and the reason.

#### Scenario: Diff adds a consumer that writes to object storage

- **GIVEN** a diff adds a Kafka consumer that uploads files to object storage with a service-account token
- **WHEN** reviewing architecture
- **THEN** the `## 2. Architecture` section contains the component findings followed by `### Reliability` (for example commit-after-durable, retries, backlog) and `### Security` (for example token handling and bucket access scope)

#### Scenario: Pure formatting change

- **GIVEN** a diff that only renames private helpers in a pure function module
- **WHEN** reviewing architecture
- **THEN** the section ends with `reliability: none` and `security: none`, each with a one-line reason

## Constraints

### Constraint: Stage boundary

The agent MUST NOT turn this stage into pure-core essays, comment nits, or a full hard-rule checklist walk.

<!-- skillet-version: 1.7.0 -->
