---
name: walk-through-behavior
description: >
  Walk through what a feature or component does before any API or code: who is involved, the journey of each
  actor as events in time order, outside systems and rules, areas grouped by business purpose, what goes in and
  out of each action, then the happy path and edge cases of each action with cause and proposed handling. The
  agent proposes each step as a plain-text diagram and explains why each part is there; the user adds, changes,
  and agrees. Based on EventStorming as shown in Anastasiia Tymoshchuk's DDD talk, and on Event Modeling. Use
  when code-buddy reaches its second stage, or when the user asks to walk through, model, or discuss the flow,
  happy path, or edge cases of a feature, service, or incident. Not for reviewing existing code (use skeptic).
spec_hash: c0dec3c5977f
---

# Walk through the behavior

Agree on what the software does, in time order, before deciding how it is built. Technology stays out until the
API is designed: no structs, crates, databases, or services in these steps, only actors, actions, events,
outside systems, and rules. Outside systems the work depends on (Kafka, S3, a payment provider) are the one
exception, because their failures become edge cases.

The agent proposes; the user decides. Every step is a draft for discussion, not a questionnaire.

## Load

1. `references/walk-through-steps.md`: the six steps, why each exists, what to show, and the light version
2. `references/notation.md`: the plain-text diagram format and the layout of `behavior.md`
3. `references/examples.md`: two walk-throughs (ice cream ordering from the talk; a Kafka snapshot consumer)

## The steps

```text
 ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
 │ 1. Who is involved  │──►│ 2. Journey per actor│──►│ 3. Outside systems  │
 │ people + systems    │   │ events in time order│   │ and rules           │
 └─────────────────────┘   └─────────────────────┘   └──────────┬──────────┘
                                                                │
 ┌─────────────────────┐   ┌─────────────────────┐   ┌──────────▼──────────┐
 │ 6. Happy path and   │◄──│ 5. In and out of    │◄──│ 4. Group into areas │
 │ edge cases          │   │ each action         │   │ by business purpose │
 └──────────┬──────────┘   └─────────────────────┘   └─────────────────────┘
            ▼
      behavior.md (agreed) ──► design the API
```

## Each step, in one message

1. **Show the draft** as a plain-text diagram (`notation.md`), built from the request and the code that exists.
2. **Explain why** each part is there: which part of the request or code it comes from, and what would be missed
   without it. Mark what is inferred.
3. **Invite the user's version:** "Do you see another actor, step, or rule? Anything to change?" The user may
   propose a different flow; discuss both and record the one agreed.
4. **Record** the agreed result in `behavior.md` right away, then stop. The next step comes in the next message.

## Edge cases (step 6)

Derive them from the flow, not from a generic list: each outside system, each gap between two steps, each rule,
each input, and each piece of shared state is a place where behavior can change (`walk-through-steps.md` § 6). For every edge
case show **Case**, **Cause**, and **Proposed handling**, marked `proposed`. List what is **Not handled** and why,
and the **Invariants** the handling protects. An edge case becomes `agreed`, and gets its Given/When/Then cases,
only when the user agrees with its handling.

## Rules

- Plain technical words. Name things by what they are ("Partition revoked between the snapshot write and the
  commit"), never by codes such as C1, E2, or D4.
- Never mark a proposal `agreed` without the user's answer. Never add an edge case, handling, or invariant to
  `behavior.md` that the user has not seen.
- One step per message. Do not batch the six steps into one wall of text.
- Diagrams in plain text or ASCII. No Mermaid or other diagram languages.
- No structs, signatures, or code in these steps. If the user asks for them, say they come after the walk-through
  (in code-buddy: design the API) and ask whether to skip ahead.
- Use the light version (`walk-through-steps.md` § Light version) when there is no state that changes over time; say why.

## Output

`behavior.md` (or the path the user names) with the agreed result of each step, in the layout of `notation.md`.
When code-buddy runs in a read-only plan mode, keep the same content in the plan file and save it to
`behavior.md` when the user leaves plan mode.
