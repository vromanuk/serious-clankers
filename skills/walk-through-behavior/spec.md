# Walk Through Behavior

## Intent

Agree with the user on what a feature or component does, in time order, before any API or code is designed. The method comes from EventStorming as presented in Anastasiia Tymoshchuk's talk "Make Your Software Architecture Sexy with Domain Driven Design": the journey of each actor, outside systems, areas split by business purpose, and what goes in and out of each use case. Event Modeling and Example Mapping add Given/When/Then cases and edge cases with their causes. The agent proposes each step as a plain-text diagram and explains why each part is there; the user adds, changes, and agrees. The result is `behavior.md`, which the API design and the tests are built from.

## Triggers

- **SHOULD** apply when code-buddy reaches its second stage, walk through the behavior.
- **SHOULD** apply when the user asks to walk through, model, or discuss the flow, happy path, or edge cases of a feature, service, or incident.
- **SHOULD NOT** apply to reviewing existing code (use skeptic) or to explaining an existing diff (use explain-diff).

## Behaviors

### Behavior: Propose and explain each step

The agent SHALL go through the six steps in `references/walk-through-steps.md` in order: who is involved, journey per actor, outside systems and rules, areas grouped by business purpose, in and out of each action, happy path and edge cases. For each step it SHALL show a draft as a plain-text diagram, explain why each part is there and where it comes from, mark what is inferred, invite the user's own version, record the agreed result in `behavior.md`, and stop. It SHALL NOT put two steps in one message.

#### Scenario: The user proposes a different flow

- **WHEN** the agent shows a journey where payment is captured when the order is placed, and the user says the shop must accept the order first
- **THEN** the agent redraws the journey with the shop's acceptance before payment, explains what changes downstream, and records the user's version

### Behavior: Edge cases come from the flow

The agent SHALL derive edge cases from places in the timeline (calls to outside systems, gaps between steps, rules, inputs, shared state, time) and SHALL show each with its case in plain technical words, its cause, and its proposed handling, marked `proposed`. It SHALL list cases considered and not handled with the reason, and the invariants the handling protects. It SHALL NOT present a generic checklist of questions for the user to fill in.

#### Scenario: Two steps are separate calls

- **WHEN** the timeline writes a snapshot to S3 and then commits Kafka offsets
- **THEN** the agent shows "Partition revoked between the snapshot write and the commit" with the rebalance as its cause and a proposed handling, and asks whether to add or change edge cases

### Behavior: Light version for code without state over time

When the work has no state that changes over time, the agent SHALL say so and use the light version: happy-path steps, in and out of each action, and a table of inputs, expected results, and why each case matters.

#### Scenario: A date parser

- **WHEN** the user wants to walk through a function that parses dates
- **THEN** the agent explains that nothing depends on earlier events and shows a table of example inputs and expected results instead of timelines

## Constraints

### Constraint: The user agrees

The agent MUST NOT mark an edge case `agreed`, or add an edge case, handling, or invariant to `behavior.md` that the user has not seen.

### Constraint: No technology before the API

The agent MUST NOT show structs, signatures, or code during the walk-through. Outside systems the work depends on may be named.

### Constraint: Plain names

The agent MUST name actors, actions, events, and edge cases in plain words, and MUST NOT label them with codes such as C1 or E2.

### Constraint: Plain diagrams

The agent MUST NOT draw diagrams in Mermaid or another diagram language; it draws them as plain text or ASCII.

<!-- skillet-version: 1.8.0 -->
