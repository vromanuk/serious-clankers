# Explanation format

Each step of the build is two messages. The first builds understanding and ends in one decision; the second shows
the code. The user decides before seeing code, and should be able to explain the piece to someone else after
reading both.

How to explain from first principles, how to pick a small example, and how to compare alternatives come from
`../../explain-diff/references/shape.md` (sections Intuition and Alternatives). This file holds only what is
specific to building code with the user.

## The question

In this order:

1. **Where we are.** The overall diagram from the build plan with this piece marked, so the user sees how it fits
   the parts already built and the parts still to come.
2. **The problem, from first principles.** What must be true, which constraint forces it, what breaks without it.
   Start from the real need in `behavior.md`, not from the code. A real, familiar case; no invented metaphor.
3. **A small example.** Toy data in and out, or a short timeline of events, using the names in `behavior.md`.
4. **Skeptic rules that apply.** The stages and references loaded for this kind of piece
   (`writing-with-skeptic.md`) and, in one line each, the rules that shape this decision.
5. **Alternatives.** Two or three real options, recommended first, in the format of `decision-triggers.md`: what,
   pros, cons, gain, when it is worse, which skeptic rule it meets or breaks, and how costly it is to change later.
   One diagram per option when the options differ in flow, states, ownership, or where code runs.
6. **"Your call?"**, and nothing after it.

Skip the question only when the build plan already records every decision for this step; say so in one line and
go to the code step.

## The code step

In this order:

1. **Skeptic rules used.** The stages loaded before writing, and what each one made the code do.
2. **Tests first,** from the agreed Given/When/Then cases in `behavior.md`: each test's name and the case it
   checks.
3. **The code** for this piece only, about 60 lines. Split a bigger piece into two steps.
4. **How it works:** the lines that matter and anything non-obvious, in a few bullets. Do not narrate every line.
5. **Edge cases:** each one handled here, with the line in `behavior.md` it comes from.
6. **Branches not in the behavior notes:** every `if` or `match` arm in decision code with no matching
   Given/When/Then case or decision. Each is either removed or raised as a decision. "None" when there are none.
7. **Choices I made without asking:** names, imports, derives, local structure. "None, because …" when there are
   none. Anything with a real choice in it is not listed here; it stops as a question.
8. **Checks:** format, lint, and test results as they came out; the skeptic stages run on this piece and what each
   one changed.
9. **"Continue?"**, and nothing after it.

## Diagrams

- At least one diagram in each question: a flow (call order), states and transitions, ownership (who holds what
  and who frees it), or where code runs (which thread or task).
- Plain text or ASCII only. No Mermaid or other diagram languages.
- Keep each diagram to about 15 lines. Label each arrow with what moves along it.
- Use the names from `behavior.md` and the code.

## Language

Simple, technical, and literal. A developer who knows the language but not this library or this codebase should
follow every sentence on the first read.

- **Literal verbs.** Say what the code does: returns, stores, calls, waits, copies, frees. Not figures of speech
  such as "hands back", "packs", "lives", "plumbs", "wires up", "in flight".
- **Say what a thing is before its library name.** "a number we attach to each record, which librdkafka returns
  with that record's result (librdkafka calls it the opaque)", not "its opaque value".
- **One name per thing, the name in the code.** Use the type, field, or function name, and say what it is the
  first time.
- **Plain names, no codes.** "The decision about where send results go", not "D4"; "Partition revoked between
  the snapshot write and the commit", not "C1".
- **Why for each step.** When the code does something unusual (splits a number into bits, skips a lock), say what
  it makes possible or what it avoids.
- **One idea per sentence**, short sentences. Well-known terms (mutex, semaphore, callback) are fine; coined labels
  and unexplained abbreviations are not.
- Say what was checked and what is assumed.

Before and after:

```text
Unclear: librdkafka hands back one usize per record (its opaque value). Stop 4 packs two things into it:
         the send's key in the high 32 bits and the record's index in the low 32 bits.

Clear:   When we give librdkafka a record, we can attach a number (a usize) to it. librdkafka never reads
         this number; it returns it to us with that record's delivery result. librdkafka calls it the opaque.
         We store two numbers in it: the upper 32 bits hold the slot index of the send in
         UnacknowledgedSends, and the lower 32 bits hold the record's position in that send's slice.
         So when a result arrives, the callback finds the right send and marks the right record
         without a separate lookup table.
```

## Example (shortened)

The question:

````text
## BufferMemory: how a send waits for room

Where we are:
  send ──► [BufferMemory] ──► UnacknowledgedSends ──► librdkafka ──► callbacks ──► SendFuture
               ^ this step

The problem: librdkafka keeps every record we give it in memory until the broker acknowledges it. If the broker
is slow, memory grows without limit and the process is killed. So something must count the bytes given to
librdkafka and not yet acknowledged, and make a send wait when the count reaches buffer_memory_bytes.

Example: limit 1,000 B. Three sends of 400 B: the first two pass (800 B), the third waits. The first is
acknowledged (400 B back), the third passes (800 B).

Skeptic rules that apply: skeptic-async (cancel safety of a waiting send; std or tokio primitives),
skeptic-reliability (a limit on memory per unit of work), skeptic-complexity (do not re-implement fair waiting).

Alternatives:
A. A tokio Semaphore whose permits are bytes (recommended)
     send ──acquire_many(300)──► [Semaphore 64 MiB] ◄──add_permits(300)── poll thread, on acknowledgment
   + fair waiting and cancel safety come with it   − permits are u32, so one send is limited to 4 GiB
   meets: skeptic-complexity (no re-implemented waiting)
B. Mutex<usize> plus Notify
     send ──lock, check, wait on Notify──► [count] ◄──lock, subtract, notify── poll thread
   + no permit size limit   − re-implements fair waiting; a cancelled wait can miss its wake-up
   breaks: skeptic-async (cancel safety needs extra care)
Recommend A: the send size limit (4 GiB) is far above any record batch. Change later: easy (private).

Your call?
````

The code step:

````text
Skeptic rules used: skeptic-async made `reserve` document its cancel safety; skeptic-naming renamed `free`
to `free_bytes`.

Tests: reserve_waits_until_release_frees_room (the 400 B example above);
send_larger_than_limit_is_rejected (edge case "send larger than the whole buffer").

```rust
... ~40 lines ...
```

How it works:
- `reserve` takes `bytes` permits and calls `forget` on them, so the caller keeps no guard object while
  librdkafka still holds the records.
- `release` adds the permits back. It runs on the poll thread and never blocks.

Edge cases: send larger than the whole buffer → rejected before `reserve` (behavior.md, "Send larger than the
whole buffer").
Branches not in the behavior notes: none.
Choices I made without asking: none, because the names and the primitive were decided above.
Checks: fmt ok, clippy ok, 6 tests passed.

Continue?
````
