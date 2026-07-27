# How to write code (detail)

Load when implementing or reviewing code shape — not required for pure Q&A.  
Same skill: `comments.md`, `design-headers.md`, `rust-craft.md`.  
Sibling skills: `skeptic-testability/references/testing.md`, `skeptic-hard-rules/references/hard-rules.md`.

## Optimize for the reader

When choosing between a short clever form and a longer plain form, pick the one a stranger can parse under pressure. Evidence of intent at the use site (names, types, decision enums, a one-line hazard comment) beats “read the implementation of every helper.”

Does **not** excuse verbose noise or restating the next line in comments.

## Project / component layout (default)

**Default:** job-shaped components with a clear public surface at the component root (Rust: `mod.rs` / `lib.rs` re-exports) and private implementation underneath; compose at the app edge. Not app-wide layer folders as the primary structure. Folders named `api/` / `internal/` are **optional**, not required.

Full rules and examples: `skeptic-architecture/references/components.md`. Always-on short form: pack `context/AGENTS.md` § Project layout.

## Performance when it matters

- Default: clarity and consistency over micro-optimization.  
- Exception: real hot paths (often Rust systems, DB, network, query engines). Then a less pretty or less “consistent with the rest of the crate” form can be right.  
- Requirement: leave **local reasoning** intact — why this is faster, what invariant the fast path needs, what a future cleaner rewrite would break.  
- Prefer measuring (bench, metrics) before large readability sacrifices.

## Delivery (chunking)

Expand here when planning multi-step work.

**Why chunk (in the work or in git history)**

- Smaller units are easier to review, revert, and bisect.  
- Feedback comes earlier; wrong direction costs less.  
- History tells a story (one idea per commit) instead of one opaque blob.

**Two valid shapes**

| Shape | When |
|-------|------|
| **Land/hand off per chunk** | I want review between steps, or pieces can ship independently |
| **One branch/PR, many commits** | Fine for a larger theme — as long as commits are meaningful and ordered |

**What to avoid:** one giant commit (or squash) that mixes many unrelated ideas with no stepwise history.

**What counts as a good chunk / commit**

| Good | Bad |
|------|-----|
| One behavior + tests that prove it | Feature + drive-by rename of half the crate in the same commit |
| One type/boundary fix with its call sites | “Types + async + docs + generate parallel” as a single commit |
| One doc or one skill | Entire harness rewrite as one squash |
| Coherent tree after the commit when practical | “Broken until the last commit” without agreement |

**How to work**

1. List chunks and order (dependency: types before call sites, pure core before shell, etc.).  
2. Implement chunk 1 → verify → **commit** (message states the one idea).  
3. Chunk 2 → … Pause for review between commits if I asked; otherwise keep going on the same branch is OK.  
4. If a chunk grows past “one clear sentence of intent,” split the commit again.

**Not** an excuse for useless micro-commits (“fix typo” × 40) or for leaving the branch uncompilable between steps without agreement.

## Thinking code vs shell

Use plain names (not pattern nicknames):

1. **Thinking code** — data in, data/decision out. No network, disk, env, clocks, random, threads, or global mutable state. Pass time in; return a **decision** for effects.
2. **Shell** — thin: read inputs, call thinking code, act on the decision.

Why: pure cores unit-test with fixed inputs; one core serves prod, bench, and tests.

Good thinking function: explicit inputs, deterministic, no side effects.

### Examples

**Compare then print**

- Thinking: `compare(s1, s2) -> Bigger | Smaller | Equal`
- Shell: read → compare → print  
- Test: `compare("b", "a")` is `Bigger`

**Update record**

- Thinking: `update_customer(existing, new) -> DoNothing | Update(...) | UpdateAndEmail(...)`
- Shell: read DB → match decision → write/email  
- Core is not async only because DB is.

### Async

Keep thinking code sync. Shell owns async I/O.

### State machines

```text
step(state, event) -> (state, decision)   # pure
```

Prefer strong invariants over dozens of one-off scenarios.

## Design before code (non-trivial)

1. Purpose — what and why  
2. Examples — given / expected (no type line in the comment)  
3. Smallest implementation  
4. Tests that mirror examples  

Skip tiny leaves named by the caller. Never skip decisions, protocol steps, public APIs.

See also: `design-headers.md`.

## Composition (one function per task)

When a problem has **distinct jobs**, give each job its own function and a thin outer function that **composes** their results (order, glue, little or no extra business rules).

Slogan: **one function per task.**

Same idea as HTDP “Composing Functions” (wish list of tasks → one function each → main only glues):
[How to Design Programs §2.3](https://htdp.org/2026-5-28//Book/part_one.html#%28part._sec~3acomposing%29).

**Why**

- Each piece stays small enough to read, test, and fix.  
- Changing one part of the problem usually means changing one function, not hunting a monolith.  
- Composition is easy to follow when each name is a real step (opening, body, closing — not “doStuff1”).

**Pseudocode — compose, don’t monolith**

```text
# Good: main composes results of clear tasks
letter(first, last, signature):
  return join(
    opening(first),
    body(first, last),
    closing(signature))

opening(first):  return "Dear " + first + ","
body(first, last):  return ... use first and last ...
closing(signature): return ... sign-off with signature ...

# Bad: one function does every subtask inline
letter(first, last, signature):
  # build greeting
  # build long body
  # build sign-off
  # hard to test middle alone; hard to change one part
```

Same idea for non-string pipelines: parse → decide → format; or shell: read → thinking → act.

**Balance with splitting**

- Split on **real tasks**, not on every line.  
- A one-call helper that only renames a loop or forwards fields is still not worth it.  
- “One function per task” does not cancel **optimize for the reader** or **performance when it matters** — a hot path may stay denser if measured and commented.  
- Blank lines between composed steps can help the reader; they are not a substitute for real functions.

When the pieces are pure (no IO), compose pure tasks + thin shell (see `skeptic-testability/references/pure-core.md`). Review checks composition under **skeptic-conventions**.

## When to extract

Extract on reuse, real boundary (parse/decide/plan/format), subtle/expensive behavior, or naming a real step — same spirit as composition.  
Do not extract one-call-site loop-namers or field-forwarding constructors.

## Edge cases

Handle edges the system can produce. Do not invent guards for impossible states; prefer types.

## Tests

- New behavior → new tests  
- Thinking: data in/out; shell: few integration tests  
- Assert outcomes, not “helper was called”  
- Prefer **public-contract** tests (unchanging under pure refactor)  
- Large input/state space → property / sequence + invariants  
- Stable complex output → snapshot/golden (normalize clocks/paths)  

**Full testing guidelines:** `skeptic-testability/references/testing.md`

## Comments (detail)

A comment exists only for truths **names, types, and tests** do not make cheap.

Prefer **why / who must / must not / negative space** over restating the next lines.

### Local reasoning (in-place evidence)

Goal: clear understanding **at the call site** (and at a module/type boundary) without needing to open every callee or reconstruct design from the whole graph.

- Prefer **names and types** that make intent obvious where the code is used (decision enums, newtypes, `must_use`, explicit ownership transfer).
- When that is not enough, put **evidence next to the use or API** — not only deep inside a helper the reader has not opened yet.
- Comments support the same goal: the explanation another engineer would look for **here**, under pressure.

This pairs with “optimize for the reader”: pay for clarity at the place people read, not only convenience for the author of the callee.

### Names carry the action

Prefer **verb phrases** for functions that do work (`load_profile_credentials`, `resolve_credentials`, `list_remote_parquet`). The name should make the action obvious at the call site.

If a `///` only restates what the name should have said, or is required for the reader to know the verb, **rename the function** instead of padding the comment. Comments remain for contracts, hazards, and non-obvious *why* — not for supplying a missing verb.

### No pure/shell stamps on docs

Do **not** label functions or modules “pure”, “No I/O”, “thinking code”, or “shell” in `//!` / `///`. That distinction is for **design and review** (can I unit-test this with data only?), not for the next reader’s API docs. Useless stamps: `/// Pure parse…`, `/// … No I/O.`, `/// Pure (no DuckDB).`

If something matters to callers, write the real rule in ordinary words (e.g. “does not open files; caller passes file text”) — only when that contract is not already obvious from the signature.

### Two kinds of comments

(Aligned with language-agnostic parts of [Google C++ Comments](https://google.github.io/styleguide/cppguide.html#Comments); Rust form differs.)

| Kind | Form (Rust) | Job |
|------|-------------|-----|
| **Documentation** | `//!` module, `///` on types and functions | Design and **intent** of what follows: purpose, contracts, given/expected on non-trivial decisions, public failure/no-op — what another engineer looks for **before** reading the body. Module docs state the job in plain words (and entrypoints / short flow when multi-step); do **not** require an “owns / does not own” template. |
| **Implementation** | `//` inside bodies | Justify non-obvious choices, tricky bits, hazards, “why not the obvious alternative” — what they look for **inside** the body |

Do not use documentation comments to restate the implementation line-by-line, or implementation comments to dump API docs that belong on the public signature. If the code is unclear, prefer a clearer rewrite when cheap; comment the remaining hazard.

### Write when real and not forced by types

1. Ownership and boundary (including what this is *not*)  
2. Caller/host obligations  
3. Identity and caching hazards  
4. Ordering and concurrency  
5. Negative space (what a copy omits and why)  
6. Preconditions of fast/unsafe paths  
7. Why two similar helpers both exist  
8. Public failure/no-op/async contracts  
9. Policy the code only half-shows  

Non-trivial functions: **design header** (purpose + given/expected). No type-signature lines. Prefer constraints over ticket IDs in comments; tickets only when necessary (e.g. critical bug scar), not on ordinary logic.

**Do not:** narrate control flow; comment obvious leaves; aspirational lies; force the reader into the implementation for a contract that should sit on the API.

Examples: `comments.md`, `design-headers.md`.  
Hard bans: `skeptic-hard-rules/references/hard-rules.md`.
