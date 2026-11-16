# Decision triggers

A decision trigger is a choice the user must make. When one comes up, in any stage, the agent stops, presents it,
and waits. The build plan records each answer in its decision list. One decision per message.

## Always stop for

| Kind | Examples |
|---|---|
| **Behavior** | Any case `behavior.md` does not cover: a duplicate, a late or missing event, a restart between two steps, a timeout, two calls on the same key, an empty or oversized input. Propose it as an edge case with cause and proposed handling (`../../walk-through-behavior/references/walk-through-steps.md` § 6). |
| **Default values and constants** | Buffer sizes, timeouts, batch sizes, retry counts, limits. Each needs its reason and what goes wrong if it is much lower or higher. |
| **Concurrency** | Mutex, channel, actor, or atomic; std or tokio mutex; spawn a task or run inline; the size of a bounded queue. |
| **State** | Typestate (the caller drives the steps), an enum of states (events decide), or plain fields; what a rejected transition returns. |
| **Simple or fuller** | The smallest version that meets the need, or a fuller one that covers more cases. |
| **Skip, defer, or build** | A feature, a check, an edge case, a test, a method nobody calls yet. |
| **Public API** | Names, signatures, owned or borrowed inputs, error variants, which states the types make impossible. |
| **Dependencies** | A new crate, a new feature flag on an existing one. |
| **`unsafe`** | Any `unsafe` block. |
| **Performance trade-offs** | Extra complexity for speed, fewer allocations, a different data structure. Say whether it is measured. |
| **Departures** | Anything the behavior notes, the agreed API, or the build plan does not already say. |
| **Test strategy** | Table, property, snapshot, or service test, and what each proves. |
| **Files outside the change** | Build files, CI, config, other crates, docs. |
| **Problems** | An ambiguity in the request or the notes, a failing check that cannot be fixed within the agreed design, a library that does not behave as assumed. Report what was found and why it matters, then propose options. |

## Not triggers

Imports, derives, re-exports, plain getters, and local names that follow the naming rules. List them under
"Choices I made without asking" in the next code step, so nothing appears unannounced.

## How to present a decision

In the question format of `explanation-format.md`. The decision part:

1. **What is being decided, and why it matters**, in one or two plain sentences.
2. **A diagram**, when the options differ in flow, ownership, or states.
3. **Options, recommended first.** For each: what it is, pros, cons, what it gains, when it is worse, which skeptic
   rule it meets or breaks.
4. **Why the recommendation, and why not the others**, in a sentence or two.
5. **How hard it is to change later:** easy (private, one place) or costly (public API, data format).
6. Stop and wait. "Do nothing" is not an option.

Example:

```text
Where send results go

Each send's result has to reach the caller. This decides what runs on librdkafka's single poll thread, which
serves every send.

  caller task ──send──► librdkafka ──► poll thread ──► ?

A. A future per send (recommended): the poll thread only wakes the caller's task; the caller's own code runs
   on its task.
   + caller code never slows other sends   − one small allocation per send
B. A callback per send: the poll thread runs the caller's code.
   + no future to hold   − one slow callback delays every send (Java and franz-go both warn about this)

Recommend A: the poll thread is shared by every caller, so caller code stays off it.
Change later: costly (public API).

Your call?
```

## Recording

Add each answer to the build plan's decision list in one line: a short title, the choice, and the reason.

```text
Where send results go: a future per send, not callbacks; keeps caller code off the shared poll thread.
```
