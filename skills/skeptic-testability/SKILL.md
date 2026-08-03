---
name: skeptic-testability
description: >
  Stage 3 of skeptic: thinking code vs shell, decisions as data, whether new
  contracts can be covered (table / property / snapshot as fit). Use when skeptic
  runs or the user asks for pure-core / testability review only. Unit-test craft
  (DAMP, unchanging, public API) is stage 4 skeptic-unit-tests — always runs
  separately.
---

# Skeptic testability

**Question:** Can decisions be tested with data in / data out, without the OS?  
**Also:** When new/changed pure contracts need coverage, is the **test shape** right (table / property / snapshot)?

**Not this stage:** unit-test *craft* (DAMP, unchanging, mock smells) — that is **stage 4** `skeptic-unit-tests` and **always** runs in the pipeline.

## Load (on demand)

- `references/testing.md` — property/snapshot shapes  
- `references/pure-core.md` — thinking vs shell samples  

## Plain words

1. **Thinking code** — data in, decision/value out; no network, disk, env, clocks, random, threads, global mutable state.  
2. **Shell** — thin: read inputs, call thinking code, act on the decision.  
3. **Decision as data** — enum/struct describing what should happen; shell performs effects.  

## Do

1. Flag business rules buried between DB/network/file/clock calls.  
2. Flag async decision logic that is async only because IO is.  
3. Prefer `step(state, event) -> (state, decision)` for long-lived state.  
4. New pure behavior → must be coverable by tests; prefer outcomes as data (not mock call counts).  
5. Prefer public-contract surfaces so tests can stay stable under pure refactor — flag pure logic only reachable via private helpers (stage 4 will flag bad tests of those helpers).  
6. **Test shape** (when new/changed pure contracts need coverage — not every diff):

   | Contract looks like… | Prefer |
   |----------------------|--------|
   | Few branches / named given-expected | Table or fixed unit cases |
   | Wide input space, state machine, always/never law | Property / sequence + invariants |
   | Stable complex blob **is** the product | Snapshot / golden (normalize noise) |

   - Do not default to property-only when a snapshot fits better (or vice versa).  
   - Flag weak generic properties when the real contract is a concrete blob.  
   - Flag snapshots of huge/noisy output without normalization.  
   - Flag missing tests when thinking code gained behavior (stage 4 also flags unit-craft gaps).  
7. Extract pure decisions when skip/error/window policy sits mid-shell.  
8. Distinct pure tasks as separate functions + thin composer when it aids testing.  

## Do not

- Full unit-test craft review (→ **unit-tests** stage) — do not skip stage 4 by doing it all here  
- Redesign whole component graph (→ architecture) unless required for testability  
- Comment form alone (→ comments)  
- Naming alone (→ naming)  
- Hard bans (→ hard-rules)  
- Production logging/spans/metrics deep-dive (→ observability)  
- Demand snapshots or properties for every small obvious branch (tables first)  

## Output section title

`## 3. Testability`
