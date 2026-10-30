# Explanation format

Each stop builds understanding from the top down: where the piece sits first, then what it does and why, then the
code. The user should be able to explain the piece to someone else after reading it.

## What each stop contains, in this order

1. **Where we are.** The overall design diagram from the brief, with this piece marked, so the user sees how it
   fits with the parts already built and the ones still to come.
2. **What it does**, in two or three plain sentences: what goes in, what comes back, what a caller sees.
3. **A diagram**, when there is a flow (call order), a state machine (states and arrows), or ownership (who holds
   what, and who frees it). Skip it when the piece is a plain function.
4. **Why this shape:** the decision, and why not the other options. Point to the decision log entries it follows.
5. **The code** for this piece only. Keep it to about 150 lines; split a bigger piece into two stops.
6. **How it works:** the lines that matter and anything non-obvious, in a few bullets. Do not narrate every line.
7. **Risks and edge cases:** what can go wrong at runtime, and how the code handles each.
8. **Tests:** each test's name and what it proves.
9. **Checks:** format, lint, and test results, reported as they came out; the skeptic stages run on this piece
   (`writing-with-skeptic.md`), and what each one made change.
10. **Open decisions**, in the format of `decision-triggers.md`, if any.
11. **"Continue?"**, and nothing after it.

## Diagrams

- Plain text or ASCII only. No Mermaid or other diagram languages.
- Draw one only when it shows something the text does not: a flow, states, ownership, or where things run (which
  thread or task).
- Keep it small, about 15 lines at most. Label each arrow with what moves along it.
- Reuse the same names as the code.

## Language

Simple, technical, and literal. A developer who knows the language but not this library or this codebase should
follow every sentence on the first read.

- **Literal verbs.** Say what the code does: returns, stores, calls, waits, copies, frees. Not figures of speech
  such as "hands back", "packs", "lives", "plumbs", "wires up", "in flight".
- **Say what a thing is before its library name.** "a number we attach to each record, which librdkafka returns
  with that record's result (librdkafka calls it the opaque)", not "its opaque value".
- **One name per thing, the name in the code.** Use the type, field, or function name, and say what it is the
  first time. Not "the send's key" when the code calls it the slot index in `UnacknowledgedSends`.
- **Why for each step.** When the code does something unusual (splits a number into bits, skips a lock), say what
  it makes possible or what it avoids.
- **One idea per sentence**, short sentences. Well-known terms (mutex, semaphore, bits, callback) are fine;
  coined labels and unexplained abbreviations are not.
- Say what was checked and what is assumed.
- Say why before how.

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

## Example stop (shortened)

````text
## Stop 7: BufferMemory

Where we are:
  send ──► [BufferMemory] ──► UnacknowledgedSends ──► librdkafka ──► callbacks ──► SendFuture
             ^ this stop

What it does: counts the bytes of records that `send` gave to librdkafka and that are not acknowledged yet,
up to buffer_memory_bytes. A send waits here when the limit is reached. When a record is acknowledged, its
bytes are added back to the count of free bytes.

  send(300 B) ──reserve──► [ 64 MiB budget ] ◄──release(300 B)── poll thread, on acknowledgment
                  waits while full

Why this shape: a semaphore whose permits are bytes (D6). A Mutex<usize> plus Notify was the alternative; it lost
because it re-implements fair waiting that the semaphore already has.

```rust
... ~40 lines ...
```

How it works:
- `reserve` takes `bytes` permits and calls `forget` on them, so the caller keeps no guard object while
  librdkafka still holds the records.
- `release` adds the permits back. It runs on the poll thread and never blocks.

Risks: a send larger than the limit would wait forever, so `send` rejects it before calling `reserve`.
Tests: reserve_waits_until_release_frees_room; send_larger_than_limit_is_rejected.
Checks: fmt ok, clippy ok, 6 tests passed. Stages run: architecture, testability, async, comments, naming,
conventions, hard rules. Async made `reserve` document its cancel safety; naming renamed `free` to `free_bytes`.

Continue?
````
