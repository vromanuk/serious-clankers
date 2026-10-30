# Decision triggers

A decision trigger is a choice the user must make. When one comes up, during design or during the build, the
agent stops, presents it, and waits. The brief records each answer in its decision log.

## Always stop for

| Kind | Examples |
|---|---|
| **Default values and constants** | Buffer sizes, timeouts, batch sizes, retry counts, limits. Each needs its reason and what goes wrong if it is much lower or higher. |
| **Concurrency** | Mutex, channel, actor, or atomic; std or tokio mutex; spawn a task or run inline; the size of a bounded queue. |
| **Simple or fuller** | The smallest version that meets the need, or a fuller one that covers more cases. |
| **Skip, defer, or build** | A feature, a check, an edge case, a test, a method nobody calls yet. |
| **Public API** | Names, signatures, owned or borrowed inputs, error variants, which states the types make impossible. |
| **Dependencies** | A new crate, a new feature flag on an existing one. |
| **`unsafe`** | Any `unsafe` block. |
| **Performance trade-offs** | Extra complexity for speed, fewer allocations, a different data structure. Say whether it is measured. |
| **Departures** | Anything the agreed design, the brief, or the spec does not already say. |
| **Test strategy** | Table, property, snapshot, or service test, and what each proves. |
| **Files outside the change** | Build files, CI, config, other crates, docs. |

## Not triggers

Imports, derives, re-exports, plain getters, and local names that follow the naming rules. Mention them in one line
at the next stop, so nothing appears unannounced.

## How to present a decision

1. **What is being decided, and why it matters**, in one or two plain sentences.
2. **A diagram**, when the options differ in flow, ownership, or states.
3. **Lettered options, recommended first.** For each: what it is, pros, cons, what it gains, when it is worse.
4. **Why the recommendation, and why not the others**, in a sentence or two.
5. **How hard it is to change later:** easy (private, one place) or costly (public API, data format).
6. Stop and wait. "Do nothing" is not an option.

Example:

```text
D4. Where send results go

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
```

## Recording

Add each answer to the brief's decision log in one line: the number, the choice, and the reason.

```text
D4. Send results: a future per send (A), not callbacks; keeps caller code off the shared poll thread.
```
