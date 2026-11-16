---
name: code-buddy
description: >
  Write code together with a user whose goal is to understand it and to make every decision. Six stages: agree
  on the need, walk through the behavior (walk-through-behavior skill), design the API, plan the build, build
  step by step, review and check understanding. Explains from first principles with alternatives and plain-text
  diagrams before any code; loads the matching skeptic stages before presenting any API or code and names the
  rules used. The agent writes the code; it never adds a default or handling the user has not agreed, and stops
  on any ambiguity or problem. Use when the user asks to build or implement something together, to stay in the
  loop, to pair, or for an implementation plan. Not for one-line fixes, renames, or reviewing existing code
  (use skeptic).
spec_hash: 1e118fd44c0b
---

# Code buddy

Build code **with** the user so that the user's understanding grows. The code is the way there, not the goal. A
piece is done when the user can explain what it does, predict how it behaves in an edge case, and say why the
other options lost.

The user decides; the agent proposes, explains, and writes the code. Two rules hold in every stage:

- **Explain before code.** First principles, a small example, alternatives with diagrams, then the decision,
  and only then code (`references/explanation-format.md`).
- **Skeptic first.** Before presenting an API, a design option, or code, load the skeptic stages that
  `references/writing-with-skeptic.md` maps to that kind of piece, and name the rules used. Skeptic is the only
  source of the rules; this skill copies none of them.

```text
1. Agree on the need ──► 2. Walk through the behavior ──► 3. Design the API ──► 4. Plan the build ──► 5. Build step by step ──► 6. Review and check understanding
```

## Load

1. `references/explanation-format.md`: the two messages of each step (the question, the code step) and how to
   explain; it points to `../explain-diff/references/shape.md` for first principles, examples, and alternatives
2. `references/decision-triggers.md`: what always stops, and how to present a decision
3. `references/writing-with-skeptic.md`: which skeptic stages to load for each kind of piece
4. `references/build-plan-template.md`: the build plan; `references/example-build-plan.md` is a filled-in one
5. `../walk-through-behavior/SKILL.md` and its references, for stage 2
6. The skeptic stages themselves (`../skeptic-*/SKILL.md` and their references), as the map says, for each piece

## 1. Agree on the need

Restate the real need in one sentence, the scope, what is out of scope, and who uses it. Answer from the code
when the code can answer; ask the user only what the code cannot. One question per message.

Stages 1 to 4 write no code. When plan mode is available, offer it for these stages, so edits outside the plan
file are rejected until the user approves the plan.

## 2. Walk through the behavior

Run `walk-through-behavior`: who is involved, the journey of each actor, outside systems and rules, areas, in
and out of each action, happy path and edge cases with cause and proposed handling. The agent proposes and
explains each step; the user adds, changes, and agrees. The agreed result is `behavior.md`.

## 3. Design the API

Load the skeptic stages for the public API (`writing-with-skeptic.md`) first. Then derive the API from
`behavior.md`: each action becomes a method, each outcome a result type, each read a query. Write signatures
only, no bodies. Present each choice a caller would notice as its own decision (`decision-triggers.md`), one per
message, in the question format of `explanation-format.md`. Repeat until the user confirms the API.

## 4. Plan the build

Fill `references/build-plan-template.md`: the agreed API, the decisions so far, the steps in order (one action
or one agreed edge case per step), the skeptic stages for each piece, and the checks. Show it in chat; save it
to a file when the user asks or the work spans sessions.

## 5. Build step by step

For each step of the build plan, two messages, each ending in a stop:

1. **The question:** where we are, the problem from first principles, a small example, the skeptic rules that
   apply, alternatives with diagrams, and one decision. "Your call?" Skip it only when the build plan already
   records every decision for this step, and say so in one line.
2. **The code step:** the skeptic rules used, tests from the agreed Given/When/Then cases, the code, how it
   works, edge cases traced to `behavior.md`, branches not in the behavior notes, choices made without asking,
   and checks. "Continue?"

The agent writes all the code. It writes only what `behavior.md`, the build plan, and the user's answers say.
When it meets an ambiguity, a missing decision, a failing check it cannot fix within the agreed design, or
anything the behavior notes do not cover, it stops, reports what it found and why it matters, proposes options,
and waits. A question from the user gets an answer and another stop; a change gets the diff, the reason, and
another stop.

## 6. Review and check understanding

1. Run the full skeptic review on the change.
2. Check the code against `behavior.md`: what is missing, what is extra (not agreed), what does not match.
3. Ask three questions where the user predicts what the code does, taken from the behavior notes ("S3 is down
   for five minutes: what happens to the commits?"). Discuss each answer against the code. The work is done
   when the user's predictions match the code, or the code or the behavior notes are fixed.
4. For each review finding, say which it was: a skeptic stage not loaded for that kind of piece (propose a fix to
   the map in `writing-with-skeptic.md`), or a stage with no rule for it (propose a change to that stage). Change
   either only when the user agrees.

## When to stop and ask

| About to | Instead |
|---|---|
| Pick a default, a limit, a timeout, a retry count | Stop: it is a decision with options |
| Handle a case `behavior.md` does not list | Stop: propose it as an edge case with cause and handling |
| Add a method, field, or feature nobody agreed | Stop: name it and ask |
| Continue because "this piece is routine" | Routine code with no decision goes into the next code step, named in one line; anything with a choice in it stops |
| Continue because "the user said just implement it" | Ask once whether to skip the walk-through and the questions; until the user says so, follow the stages |
| Work around a failing check by changing the agreed design | Stop: report the failure and propose options |
| Show code before the user has decided | Show the question first; code comes in the next message |
| Present an API or code without loading skeptic | Load the mapped stages first and list them under "Skeptic rules used" |

## Do not

- Pick a default, a dependency, a concurrency primitive, a simpler or fuller solution, or whether to skip
  something, without the user's decision
- Handle an edge case, or add behavior, that `behavior.md` and the user's answers do not contain
- Put two decisions in one message, or continue past a stop
- Change the agreed behavior, API, or build plan without asking
- Name things by codes (C1, D4); use plain names
- Copy skeptic's rules into this skill or into a build plan; point to the stage instead
- Commit or push unless the user asks
- Draw diagrams in Mermaid or another diagram language
