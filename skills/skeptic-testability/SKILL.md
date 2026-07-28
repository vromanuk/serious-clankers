---
name: skeptic-testability
description: >
  Stage 3 of skeptic: thinking code vs shell, decisions as data, tests for new
  contracts (table / property / snapshot as fit). Use when skeptic runs or the
  user asks for pure-core / testability review only.
---

# Skeptic testability

**Question:** Can decisions be tested with data in / data out, without the OS?  
**Also:** When new/changed pure contracts need coverage, is the **test shape** right?

## Load (on demand)

- `references/testing.md` — property/snapshot shapes  
- `references/pure-core.md` — thinking vs shell samples  
- `../unit-tests/SKILL.md` + `../unit-tests/references/guide.md` — **unit-test quality** (unchanging, public API, state vs interaction, DAMP)  

## Plain words

1. **Thinking code** — data in, decision/value out; no network, disk, env, clocks, random, threads, global mutable state.  
2. **Shell** — thin: read inputs, call thinking code, act on the decision.  
3. **Decision as data** — enum/struct describing what should happen; shell performs effects.  

## Do

1. Flag business rules buried between DB/network/file/clock calls.  
2. Flag async decision logic that is async only because IO is.  
3. Prefer `step(state, event) -> (state, decision)` for long-lived state.  
4. New behavior → tests; pure cores: assert outcomes, not mock call counts.  
5. Prefer public-contract tests (stable under pure refactor); flag brittle internal-only asserts.  
5b. **Unit-test craft** (load `unit-tests` guide when tests are in scope): unchanging tests; state not interactions; behaviors not methods; DAMP over DRY; clear failure messages.  
6. **Test shape** (when new/changed pure contracts need coverage — not every diff):

   | Contract looks like… | Prefer |
   |----------------------|--------|
   | Few branches / named given-expected | Table or fixed unit cases |
   | Wide input space, state machine, always/never law | Property / sequence + invariants |
   | Stable complex blob **is** the product | Snapshot / golden (normalize noise) |

   - Do not default to property-only when a snapshot fits better (or vice versa).  
   - Flag weak generic properties when the real contract is a concrete blob.  
   - Flag snapshots of huge/noisy output without normalization.  
   - Flag missing tests when thinking code gained behavior.  
7. Extract pure decisions when skip/error/window policy sits mid-shell.  
8. Distinct pure tasks as separate functions + thin composer when it aids testing.  

## Do not

- Redesign whole component graph (→ architecture) unless required for testability  
- Comment form alone (→ comments)  
- Naming alone (→ naming)  
- Hard bans (→ hard-rules)  
- Production logging/spans/metrics deep-dive (→ observability)  
- Demand snapshots or properties for every small obvious branch (tables first)  

## Output section title

`## 3. Testability`
