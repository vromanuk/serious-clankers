# Steps of the walk-through

Sources: Anastasiia Tymoshchuk, "Make Your Software Architecture Sexy with Domain Driven Design" (Python Web Conf
2022): EventStorming of user journeys, split into domains by business purpose, messages in and out per use case,
then C4. Alberto Brandolini, *Introducing EventStorming* (events in the past tense, policies, "always?
immediately?"). Adam Dymitruk, eventmodeling.org (commands, views, Given/When/Then per slice, every field traced).
Matt Wynne, Example Mapping (rules, examples, questions).

Each step is one message: draft, why, the user's version, record, stop.

## 1. Who is involved

**Show:** the people and programs that act (customer, shop manager, the consumer process, a scheduler) and the
outside systems the work depends on but does not build (Kafka, S3, a payment provider).

**Why it exists:** each actor has its own journey. A missing actor means missing behavior; it usually shows up
later as an unplanned edge case.

**Ask:** "Anyone or anything else that acts or that we depend on?"

## 2. Journey per actor

**Show:** one timeline per actor, from the first contact to the end, as actions (`>PlaceOrder`) and events
(`!OrderPlaced`, past tense) in time order. Include what the actor reads before an action (`?NearbyShops`). Keep
it business-level: "it's hot → sees an ad → installs the app → finds nearby shops → orders", not "POST /orders".

**Why it exists:** it states what the software does before anyone thinks about structure. Gaps in the story
("how does the shop learn about the order?") are found here, while they are cheap.

**Explain:** where each event comes from (the request, existing code, an assumption). Mark assumptions.

**Ask:** "Does this match how it works? A step missing or in the wrong order?"

## 3. Outside systems and rules

**Show:** where each outside system enters the timeline, and the rules that connect events to actions:
"whenever `!OrderPlaced`, `>CapturePayment`". Ask of each rule "always? immediately?": the answer often splits one
rule into two, or exposes a missing read or a missing wait.

**Why it exists:** outside systems fail, retry, and answer late, and we do not control them; each one is a source
of edge cases. Rules are where hidden logic sits; written down, they can be questioned.

**Ask:** "Is each rule always true, and does it happen at once or later?"

## 4. Group into areas

**Show:** events grouped by business purpose (sign-up, discovery, ordering, delivery), with the arrows between
groups drawn. A group does not have to be one contiguous block of the timeline.

**Why it exists:** a group with one purpose and few arrows to others becomes one component with a small API.
Many arrows between two groups means the split is wrong: move events until the arrows are few.

**Explain:** for each group, its purpose in one line and the arrows that leave it.

**Ask:** "Would you split these differently?"

## 5. In and out of each action

**Show:** each action and each read as a black box: what goes in, what comes back, what the caller sees on
failure. From the talk: "Locate nearby shops. In: user id, latitude, longitude. Out: a list of shops."

**Why it exists:** this is what the public API is made from. Methods come from actions, result types from
events, queries from reads. Nothing is invented in the API design that is not here.

**Ask:** "Anything missing from what goes in or what comes back?"

## 6. Happy path and edge cases

**Show,** for each action:

```text
Use case: <action in plain words>

  Happy path
    <steps in one or two lines>

  Edge cases
    <case, in plain technical words>
      Cause: <why it can happen, pointing at the timeline>
      Proposed handling: <what the code does>      [proposed]

  Not handled
    <case>: <why it cannot happen or does not matter>

  Invariants
    <a rule that must always hold>

  Edge cases to add or change?
```

**Where edge cases come from.** Walk the timeline and look at each place behavior can change. Show only the cases
that apply, each with its cause; list the ones considered and left out under Not handled:

| Place in the timeline | Edge cases it can produce |
|---|---|
| A call to an outside system | timeout, error reply, reply lost after the work was done, slow reply, retry by the other side (duplicate) |
| The gap between two steps | crash or restart between them, ownership moved (rebalance, lease lost), state changed by someone else |
| A rule ("whenever X, do Y") | X arrives twice, late, out of order, or never; Y fails after X was recorded |
| An input | empty, too large, malformed, from an unknown or unauthorized caller |
| Shared state | two calls change the same key at once; a reader sees a half-finished change |
| Time | a deadline passes, a clock jumps, a window closes while work is still open |

**Why it exists:** an edge case decided here is a decision the user made. An edge case found while writing code is
a decision the agent would otherwise make alone.

**Agreeing:** an edge case moves from `proposed` to `agreed` only on the user's answer. Then add its
Given/When/Then cases (`notation.md`); each becomes one test when the code is built.

## Light version

Use it when there is no state that changes over time (a parser, a pure calculation, a one-shot script), and say
why. Steps 1–4 become one short list of happy-path steps. Step 5 stays. Step 6 becomes a table:

```text
input                         expected                     why this case
""                            Err(Empty)                   empty input is allowed by the caller
"2026-02-30"                  Err(InvalidDate)             day does not exist in that month
```

Move to the full steps when the expected result starts to depend on what happened before: that is state over time.

## Finish

Show the whole `behavior.md` once: the timeline, the areas, in and out of each action, and every edge case with
its status. Ask whether anything is still open. Proposed edge cases left open are listed as open questions for
the API design.
