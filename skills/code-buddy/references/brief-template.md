# Brief template

The brief is the prompt an agent follows to build the agreed design with the user in the loop. It holds what is
specific to this task: the design, the decisions, the stop points, and which skeptic stages apply to which piece.
It does not copy skeptic's rules; it points to them, so a brief never goes stale when a stage changes.

Fill every `<…>`. Drop a section only when it cannot apply. `example-brief.md` is a filled-in brief.

````text
You are building <what> with the user. Read this whole brief before writing code.

## How the user works
- Top-down, by wishful thinking: the behavior they wish they had comes first, as calls to pieces that do not exist
  yet; each piece's inside comes later, one at a time. Never start from the bodies.
- The user must understand every piece as it is built: the high-level picture first, why each decision and not
  the other options, simple diagrams where they help.
- The user makes every decision: defaults, concurrency primitives, simple or fuller, skip or build. Propose
  options and wait; never pick silently.

## Context
- Repo and branch: <path>, <branch>. Crate or module: <name>.
- Requirements: <spec path, or "none">. Build against them; never change them to match the code.
- Code to imitate: <files whose style, docs, and shape this code follows>.
- Do not commit or push unless the user asks. <Other limits: files not to touch, docs not to write.>

## Agreed design
<The public face: job struct, methods with signatures, public data types and where they live.>
<The private parts: each piece, what it holds, its methods.>
<Concurrency: who calls what from where, what is shared, what runs on which thread or task.>
<Defaults and constants: each value, decided.>
<Anything else decided: allocations, dependencies, what is out of scope.>

## Decision log
<D1. <choice> (<option letter>), not <other>; <reason in one line>.>
<D2. …>

## Stay in the loop
Stop at each stop point below. Never put two in one message, never continue because a step looks routine.
Stop points:
1. The public face: types and signatures, no bodies.
2. Every default and constant, each with its reason and what goes wrong much lower or higher.
<3… one per critical piece: state machines, concurrency, error mapping, configuration, shutdown, metrics.>
<n. Any change to files outside <scope>.>
<n+1. Anything outside the agreed design, any new dependency, any performance trade-off, any `unsafe`: ask first.>

Each stop contains, in order: where we are (the design diagram with this piece marked); what it does; a diagram
when there is a flow, states, or ownership (ASCII, no Mermaid); why this shape and why not the other options
(point to the decision log); the code for this piece only (about 150 lines at most); how it works; risks and edge
cases; tests and what each proves; checks (format, lint, tests, and the skeptic stages run); open decisions as
lettered options with the recommendation first. End with "Continue?" and wait. A question gets an answer and
another stop; a change gets the diff, the reason, and another stop.

Decisions not in the decision log always stop: defaults, concurrency primitives, simple or fuller, skip or build,
public API, dependencies, `unsafe`, performance trade-offs, test strategy, files outside the change.

## Build order
<types -> rules (no IO) -> IO parts -> wiring -> metrics -> service tests, or this task's own order>

## Writing with skeptic
Skeptic is the source of the rules. Before writing each piece, load the stages for it, follow them while writing,
then run their questions on the piece and fix what fails before showing it.
Stages live in ~/.agents/skills/<stage>/SKILL.md, with their references.
- Every piece: skeptic-naming, skeptic-comments, skeptic-hard-rules.
<- <piece>: <stages>, e.g. "UnacknowledgedSends: skeptic-testability (pure-core.md), skeptic-complexity">
<- …>
Read the Scars list in skeptic/SKILL.md before the first piece.

## Checks after each piece
<format command>; <lint command>; <test command>. Report the results as they came out.
````
