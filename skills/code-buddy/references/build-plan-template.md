# Build plan template

The build plan is the prompt an agent follows to build the agreed behavior and API with the user in charge of
every decision. It holds what is specific to this task: the behavior notes, the API, the decisions, the steps,
and which skeptic stages apply to which piece. It does not copy skeptic's rules; it points to them, so a build
plan never goes stale when a stage changes.

Fill every `<…>`. Drop a section only when it cannot apply. `example-build-plan.md` is a filled-in build plan.

````text
You are building <what> with the user. Read this whole build plan, and behavior.md, before writing code.

## How the user works
- The goal is the user's understanding. Explain before code: where we are, the problem from first principles, a
  small example, the skeptic rules that apply, alternatives with diagrams, then one decision. Code comes in the
  next message.
- The user makes every decision. You write the code, and only what behavior.md, this plan, and the user's answers
  say. No default, no limit, no handling, no method that is not agreed.
- On any ambiguity, missing decision, or problem: stop, report what you found and why it matters, propose
  options, and wait.

## Context
- Repo and branch: <path>, <branch>. Crate or module: <name>.
- Behavior notes: <path to behavior.md>. Build against them; never change them to match the code.
- Requirements: <spec path, or "none">.
- Code to imitate: <files whose style, docs, and shape this code follows>.
- Do not commit or push unless the user asks. <Other limits: files not to touch, docs not to write.>

## Agreed API
<The public API: job struct, methods with signatures, public data types and where they live. Each method names
the action in behavior.md it comes from.>
<The private parts: each piece, what it holds, its methods.>
<Concurrency: who calls what from where, what is shared, what runs on which thread or task.>
<Defaults and constants: each value, decided.>
<Anything else decided: allocations, dependencies, what is out of scope.>

## Decisions so far
<Short title: choice, not <other option>; reason in one line.>
<…>

## Steps
One action or one agreed edge case per step, in this order:
<1. Types: the public API, no bodies.>
<2. … each step names the behavior.md cases it builds.>

Each step is two messages. The question: where we are, the problem, a small example, the skeptic rules that
apply, alternatives with diagrams, "Your call?" (skip it, saying so in one line, when this plan already records
every decision for the step). The code step: skeptic rules used, tests from the Given/When/Then cases, the code
(about 60 lines), how it works, edge cases traced to behavior.md, branches not in the behavior notes, choices
made without asking, checks, "Continue?". Never put two decisions in one message; never continue past a stop.

Decisions not in this plan always stop: behavior not in behavior.md, defaults, concurrency primitives, simple or
fuller, skip or build, public API, dependencies, `unsafe`, performance trade-offs, test strategy, files outside
the change, and any problem you cannot solve within the agreed design.

## Writing with skeptic
Skeptic is the source of the rules. Before presenting each piece, load the stages for it and name them; follow
them while writing; run their questions on the piece and fix what fails before showing it.
Stages live in ~/.agents/skills/<stage>/SKILL.md, with their references.
- Every piece: skeptic-naming, skeptic-comments, skeptic-hard-rules.
<- <piece>: <stages>, e.g. "UnacknowledgedSends: skeptic-testability (pure-core.md), skeptic-complexity">
<- …>
Read the Scars list in skeptic/SKILL.md before the first piece.

## Checks after each step
<format command>; <lint command>; <test command>. Report the results as they came out.

## At the end
Full skeptic review; check the code against behavior.md (missing, not agreed, does not match); three questions
where the user predicts what the code does, taken from behavior.md.
````
