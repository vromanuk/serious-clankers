---
name: skeptic
description: >
  Coordinate a multi-stage code review in fixed order: purpose, architecture,
  complexity, testability, unit-tests, observability, comments, naming,
  conventions, hard-rules. Use when the user asks for review, code review, comprehensive
  review, review this branch/PR/diff, or /skeptic. Prefer for Rust codebases.
  Not for post-implement fix-only loops, file-by-file progressive campaigns, or
  external automated-review CLIs alone.
---

# Skeptic

Standalone, **read-only** multi-stage code review, **Rust-first**. Snapshot scope and real need, run stages 1→10, judge findings, merge one report, ask before any fix.

## Contract

- **Read-only** unless the user explicitly asked to fix (fix mode, Loop step 4.4).
- Run stages **1→10 always**, in report order. Never freestyle a product essay first. **Do not skip** unit-tests (stage 5) — it is not optional depth under testability.
- **One stage at a time, over a fixed file scope.** Each stage gets its own pass over every scoped file with only that stage's question in mind. A single pass that tries to apply all ten lenses at once skims and misses findings that a dedicated pass catches.
- **Execution:** run the full review **in this same session**. Do not spawn subagents. Load each stage `SKILL.md`, keep stage boundaries, same report shape. Do not invent a one-lens freestyle essay.
- **Coordinator:** reject weak, preference-only, or evidence-free findings; label facts vs assumptions.
- Findings: numbered issues with evidence; **LETTER options** only for material design forks (real alternatives — no “do nothing”). Per option: what / pros / cons / gain / worse when. No time estimates.
- Ask before implementing fixes, unless the user asked to fix (fix mode).
- Deterministic tools (`rustfmt`, `clippy`, tests) are validation notes — not stages and not taste debates.

## Stages (always)

| # | Stage skill | Question |
|---|-------------|----------|
| 1 | `../skeptic-purpose/SKILL.md` | Real need? Approach fit? Serious bugs? Alternatives? |
| 2 | `../skeptic-architecture/SKILL.md` | Default layout: job components? Use-case surface (not stray helpers)? Data ownership? Types at boundaries? Plus `### Reliability` (`../skeptic-reliability/SKILL.md`) and `### Security` (`../skeptic-security/SKILL.md`) when the diff touches failure-relevant code or a trust boundary |
| 3 | `../skeptic-complexity/SKILL.md` | Harder to understand or change? Shallow units, leakage, pass-throughs, error design, split vs join? |
| 4 | `../skeptic-testability/SKILL.md` | Thinking vs shell? Decisions as data? Coverage/shape for new contracts? |
| 5 | `../skeptic-unit-tests/SKILL.md` | Unit-test craft: public API, state not mocks, DAMP, unchanging? |
| 6 | `../skeptic-observability/SKILL.md` | Logs/spans/metrics useful? Async spans? Safe labels? Alerts: precision, recall, detection, reset, how many fire? |
| 7 | `../skeptic-comments/SKILL.md` | Comments: necessary? why not what? clear English? |
| 8 | `../skeptic-naming/SKILL.md` | Names: clear for scope, not cryptic, not overlong? |
| 9 | `../skeptic-conventions/SKILL.md` | One function per task? Plain words? Clear Rust style? |
| 10 | `../skeptic-hard-rules/SKILL.md` | Absolute bans only (`references/hard-rules.md` on that skill)? |

Each stage owns its own `references/` (load only what that stage’s SKILL asks for). Coordinator stays thin — no shared ref library here.

**Extra section (conditional):** when the diff contains async Rust (`async`, `.await`, `tokio::`, futures, a hand-written `Future`), also run `../skeptic-async/SKILL.md` after stage 10 and report it as `## Async`. No async code → omit the section.

**Extra section (conditional):** when the diff adds or changes traits, trait implementations, types with behavior, module layout, crate boundaries, `Cargo.toml` dependencies, or a library crate's public API, also run `../skeptic-solid/SKILL.md` after stage 10 (after `## Async` when present) and report it as `## SOLID`. Otherwise omit the section.

## Care priority (when weighting findings)

1. Correctness of the real contract  
2. Testability of decisions  
3. Clarity (names, functions that each do one thing completely, plain comments)  
4. Small surface  
5. DRY on meaning  
6. Performance last by default  

## Scars (flag when the diff shows them)

- Layer soup / package theater for a one-shot script  
- Shallow component: many public step-helpers; callers reassemble the use case  
- Function split for length so the pieces must be read together (conjoined)  
- Error no caller can act on, where redefining the operation would remove it  
- Free public orchestration fns that re-pass the same deps (prefer a job struct)  
- Handler owns multi-step orchestration that belongs on the component struct  
- Business rules next to sockets/files/clocks  
- Speculative guards with no contract  
- Restating comments / type lines in design headers  
- Unclear or overlong names for how widely they’re used  
- Function named as a noun, or a verb reused as a noun (`resume_offsets` for the next offset to fetch)  
- Ambiguous short variable (`list`, `stale`, `read`) that does not say which thing  
- Type or value named for the episode or the decision (`UnfinishedAfterRevoke`, `IgnoredFinish`) instead of the thing (`UnfinishedRecords`)  
- New struct that repeats an existing struct or enum variant under new field names, instead of reusing or generalizing it  
- Public data type (`struct` / `enum` / `type`) declared outside the crate’s `types` module  
- Helpers that only rename a loop; big functions packing several jobs  
- Import of another component’s private surface  
- Shared write tables across components  
- Hand-rolled date/time when a battle-tested lib fits  
- One opaque mega-diff / squash of many ideas  
- Contract only in a second helper — caller can forget  
- `println!` as production logging; unentered spans; metric labels with unbounded values  
- Alert that pages on a cause, filters noise only with a long `for:`, or fires once per pod for one incident  
- External HTTP/DB/RPC call with no timeout or deadline  
- Brittle unit tests (private helpers, interaction-only mocks, case hidden in test helpers)  
- Public API that exposes implementation units (upload slots, parts, files, provider types), so swapping the implementation changes callers  
- Per-record bookkeeping (ids, tracker entries) left out of the memory limit that pauses input  
- CPU-heavy work (encoding, compression, hashing large buffers) on async runtime threads  
- Collaborators built inside a component instead of passed in from the app edge  

Hard-rule IDs only from stage 10 / `skeptic-hard-rules/references/hard-rules.md`. Soft scars go to the matching stage with evidence.

## Concern format

```text
[severity][evidence:<label[,label]> <locator>] path:line — concern. impact: <impact>. fix: <smallest change or "ask user">.
```

Severity: `blocker` | `high` | `medium` | `low`  
Evidence: `direct` | `spec` | `policy` | `test` | `validation` | `missing` | `inferred`  
(`inferred` never sole label on `blocker`)

Material design forks: NUMBER the issue, then LETTER real options (recommended first).

## Loop

1. **Snapshot** — base/dirty tree, real need (mark assumed if needed), non-goals, changed files, validation commands available  
2. **Scope files** — list the files that carry the change: production code first, then its tests. For a shared file the change only touches, scope the diff, not the whole file. Leave out build and generated files (BUCK, lockfiles, generated code, deploy templates) unless the change is about them. State the list to the user before stage 1. Read every scoped production file in full once; read large test files in full when stage 5 runs.  
3. **Track progress** — one task per step: scope, stages 1–10, `## Async` / `## SOLID` when they apply, final checks. Mark each done as it finishes, so the user sees which stage is running.  
4. **Run each stage on its own**, in order 1→10, then `## Async` and `## SOLID` when their triggers match. In this session, no spawned agents. For every stage:  
   1. Load that stage's `SKILL.md` and each reference it says to load — read the sections it names, not only the headings.  
   2. Walk every scoped file with only this stage's question. Re-open the code at the lines in question instead of relying on memory of an earlier pass.  
   3. Write this stage's findings (or `ok` / `none` with a one-line why) before starting the next stage. A concern that belongs to a later stage waits for that stage.  
   4. **Fix mode** (the user asked to fix or address issues): fix this stage's findings that keep the intent, then run the smallest check that covers them (build, clippy, the touched tests) before the next stage. A finding with a real design fork is asked about or listed as open — never picked silently. When a fix adds a test for a contract, break the code once to see the test fail, then restore it.  
   Do not skip a stage because the slice “looks safe.”  
5. **Coordinate** — accept evidence-backed stage-appropriate concerns; dedupe; facts vs assumptions  
6. **Validation** — review-only: smallest relevant checks, optional, not a tenth stage. Fix mode: fmt, clippy and the full relevant test suites once after the last stage; name tests that cannot run here and why.  
7. **Handoff** — one merged report; do not implement unless asked  

One full pass unless the user asks for re-review after fixes. Not an implement→review loop.

## Handoff report

```text
# Skeptic — <scope>

Scope: <files reviewed; what was left out and why>
Checked: …
Assumed: …
Real need: …

## 1. Purpose
## 2. Architecture
## 3. Complexity
## 4. Testability
## 5. Unit tests
## 6. Observability
## 7. Comments
## 8. Naming
## 9. Conventions
## 10. Hard rules
## Async (only when the diff has async code)

## Validation
(commands/results or not run)

## Recommended next steps
(ordered by gain and risk of leaving as-is; ask what to implement)
```

In fix mode, each stage section lists finding → fix, and a closing `## Open decisions` lists the design forks left for the user (numbered issues, lettered options).

Final line: `skeptic: complete` or `skeptic: blocked` (scope/need missing).

## Do not

- Freestyle product essay that skips the ten stage contracts  
- Several stages in one pass over the files, or a stage judged from memory of an earlier pass  
- Skip unit-tests stage or fold it silently into testability  
- Spawn subagents for stages  
- “Do nothing” as a design option  
- Implement without asking  
- LLM as formatter/linter  
- Invent hard-rule IDs  
