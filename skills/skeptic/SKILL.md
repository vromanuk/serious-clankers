---
name: skeptic
description: >
  Coordinate a multi-stage code review in fixed order: purpose, architecture,
  testability, conventions, hard-rules. Use when the user asks for review, code
  review, comprehensive review, review this branch/PR/diff, or /skeptic.
  Prefer for Rust codebases. Not for post-implement fix-only loops, file-by-file
  progressive campaigns, or external automated-review CLIs alone.
---

# Skeptic

Standalone, **read-only** multi-stage code review, **Rust-first**. Snapshot scope and real need, run stages 1→5, judge findings, merge one report, ask before any fix.

## Contract

- **Read-only** unless the user explicitly asked to fix.
- Run stages **1→5 always**, in report order. Never freestyle a product essay first.
- **Execution mode (prefer subagents, never block on them):**
  - **With subagents:** one no-edit subagent per stage; at most **3** open at once; rolling spawn / wait / collect / refill; drain before handoff.
  - **Without subagents (or capacity too low):** run every stage **in this same session**, in order 1→5, still loading each stage `SKILL.md` and keeping stage boundaries. Same report shape. Do not invent a one-lens freestyle essay.
- **Coordinator** (same either mode): reject weak, preference-only, or evidence-free findings; label facts vs assumptions.
- Findings: numbered issues with evidence; **LETTER options** only for material design forks (real alternatives — no “do nothing”). Per option: what / pros / cons / gain / worse when. No time estimates.
- Ask before implementing fixes.
- Deterministic tools (`rustfmt`, `clippy`, tests) are validation notes — not stages and not taste debates.

## Stages (always)

| # | Stage skill | Question |
|---|-------------|----------|
| 1 | `../skeptic-purpose/SKILL.md` | Real need? Approach fit? Serious bugs? Alternatives? |
| 2 | `../skeptic-architecture/SKILL.md` | Default layout: job components (not layers)? Public vs private? Data ownership? Types at boundaries? |
| 3 | `../skeptic-testability/SKILL.md` | Thinking vs shell? Decisions as data? Tests for new contracts? |
| 4 | `../skeptic-conventions/SKILL.md` | Comments, plain language, composition, Rust form? |
| 5 | `../skeptic-hard-rules/SKILL.md` | Absolute bans only (`references/hard-rules.md` on that skill)? |

Each stage owns its own `references/` (load only what that stage’s SKILL asks for). Coordinator stays thin — no shared ref library here.

## Care priority (when weighting findings)

1. Correctness of the real contract  
2. Testability of decisions  
3. Clarity (names, short functions, plain comments)  
4. Small surface  
5. DRY on meaning  
6. Performance last by default  

## Scars (flag when the diff shows them)

- Layer soup / package theater for a one-shot script  
- Business rules next to sockets/files/clocks  
- Speculative guards with no contract  
- Restating comments / type lines in design headers  
- One-use loop-namer helpers; monoliths packing several tasks  
- Import of another component’s private surface  
- Shared write tables across components  
- Hand-rolled date/time when a battle-tested lib fits  
- One opaque mega-diff / squash of many ideas  
- Contract only in a second helper — caller can forget  

Hard-rule IDs only from stage 5 / `skeptic-hard-rules/references/hard-rules.md`. Soft scars go to the matching stage with evidence.

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
2. **Load** each stage `SKILL.md` (and that stage’s `references/` as the stage says). Do not skip a stage because the slice “looks safe.”  
3. **Choose mode** — subagents if available; else same-session sequential stages  
4. **Run stages 1→5** — subagent-per-stage (max 3 concurrent) **or** sequential in this window  
5. **Coordinate** — accept evidence-backed stage-appropriate concerns; dedupe; facts vs assumptions  
6. **Optional validation note** — smallest relevant checks; not a sixth stage  
7. **Handoff** — one merged report; do not implement unless asked  

One full pass unless the user asks for re-review after fixes. Not an implement→review loop.

## Stage subagent prompt

```markdown
Review only. Do not edit. Return findings only for this one stage.

Stage: <1-5 name>
Stage contract (full text of the stage SKILL.md):
<paste stage SKILL.md>

Care priority (highest first): real contract → decision testability → clarity → small surface → meaning-DRY → perf last by default.

User goal / real need: <sentence; mark assumed if needed>
Scope: <base, changed files>
Non-goals: <or none>
Intentional tradeoffs: <or none>
Validation already run: <or not run>
Stage reference paths (read if stage says so): <list from that stage’s SKILL>

Rules:
- Stay inside this stage’s Do / Do not.
- Findings need path:line (or command/artifact locator) and evidence labels.
- Hard-rule IDs only in stage 5, from hard-rules.md — do not invent IDs.
- No time estimates. No implementing.
- If no material concerns: `none` plus one-line why.

Output:
- Section title as required by the stage skill
- Zero or more concern lines in skeptic format
- For material design forks only: NUMBER + LETTER options
```

## Handoff report

```text
# Skeptic — <scope>

Checked: …
Assumed: …
Real need: …

## 1. Purpose
## 2. Architecture
## 3. Testability
## 4. Conventions
## 5. Hard rules

## Validation
(commands/results or not run)

## Recommended next steps
(ordered by gain and risk of leaving as-is; ask what to implement)
```

Final line: `skeptic: complete` or `skeptic: blocked` (scope/need missing — not “no subagents”).  
Note in the report which mode ran: `mode: subagents` or `mode: same-session`.

## Do not

- Freestyle product essay that skips the five stage contracts  
- Refuse to run only because subagents are unavailable  
- “Do nothing” as a design option  
- Implement without asking  
- LLM as formatter/linter  
- Invent hard-rule IDs  
- When subagents exist: collapse five stages into one subagent “to save time”  
