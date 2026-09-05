---
name: pr-description
spec_hash: 4392f29718ef
description: >
  Draft pull-request descriptions from committed branch changes. Use when the
  user asks to write, draft, or generate a PR description, fill a PR body, or
  summarize changes for a PR. Not for full multi-lens code review or commit
  messages alone.
---

# PR description

Write a PR body a reviewer can use. **Inspect committed branch changes first**
(changed files + diff), then draft. Plain language.

**Always the same structure** — fixed labels, same order, every time:

```text
Why: <why we need this — missing contract + short intuition>

How: <intuition for the approach, then the mechanism>

Testing: <what was run or what to run>
```

When structure matters, add a **separate** diagram/components block between `---` lines (not inside `How:`):

```text
Why: <need + intuition>

How: <approach intuition, then mechanism>

---

<diagram and/or changed components>

---

Testing: <what was run or what to run>
```

## Workflow

1. **Inspect committed changes** (required — do this before drafting):
   - Current branch; base = user-named base, else `main` / `master` / default.
   - Merge base: `git merge-base HEAD <base>`.
   - **Changed files (committed on the branch):**  
     `git diff --name-status <merge-base>...HEAD`
   - **Commits:** `git log --oneline <merge-base>..HEAD`
   - **Diff:** `git diff <merge-base>...HEAD` (full or by important paths). Read enough of the real patches to explain the change — do not invent from commit subjects alone when the diff is available.
   - Ticket: if a key is in the branch name, PR title, or user text, fetch it for the product need. Keep `Why:` short — do not paste the ticket.
   - Uncommitted dirty files: ignore for the PR body unless the user asked to include WIP; if you skip them, do not claim they are in the PR.
2. **Analyze** — from that file list + diff: why the change is needed, how the existing layout extends, mechanism, whether a diagram is needed, testing evidence.
3. **Draft** using `references/shape.md` (load it). Labels must match exactly.
4. **Check** — grounded in committed files/diff; `Why:` → `How:` → optional `---` diagram → `Testing:`; no invented runs; `Why:` is not a file dump.
5. **Deliver** the full body (and put it in the PR if the user asked to open/update one).

## Fixed structure (mandatory)

| Label | Job |
|-------|-----|
| `Why:` | Why we need this: named missing value, so the caller cannot fetch it, plus a short intuition |
| `How:` | Intuition for the approach (existing pattern this extends), then the mechanism |
| `---` … `---` | Diagram / changed components when structure matters |
| `Testing:` | Commands, cases, or honest manual check |

Optional after those when material: `Risks:` or `Notes:`.

### Rules

- Start lines with exactly `Why:`, `How:`, `Testing:`.
- **Do not** replace them with `## Summary` or freeform headings, or reorder them.
- Tiny one-liner PRs still use the same labels. No empty `---` pair.

### Why (need + intuition)

`Why:` explains **why this change exists**. Technical names, short sentences. Fold the intuition into the paragraph — do not add a second heading.

1. **What is not there** — signal, field, row, in the system's names.  
2. **So the caller cannot fetch it** — which RPC/store lookup fails (empty even though producers send it).  
3. **Who needs it** — one line + ticket if fetched. Label *assumed* if only inferred from the diff.

Not a file dump. Not the implementation (that is `How:`).

### How (approach intuition, then mechanism)

First: how we are going about it — the existing layout this extends (e.g. one ingest deployment per product; numbers already served elsewhere). Then: the mechanism in a few sentences (ingest path, keep policy, routing).

Not: env-var / metric / clone inventory, “please review” lists, vague jobs (“for restart” → say what the component actually does). No diagram in this paragraph.

### Diagram / changed components

- Multi-component or boundary change → sketch and/or short component list **between** two `---` lines (after `How:`, before `Testing:`).
- Tiny single-site fix → omit the whole `---` block.

### Testing

- Name commands, test names (especially tests **in the committed diff**), or honest manual steps.
- Never bare `N/A` / `tested` without what was checked.

### Accuracy

- Only claim what committed files, the branch diff, and session evidence support.
- Do not invent features, services, files, or CI runs.

## Voice

- Direct. Technical names, human sentences. No corporate fluff.
- Short. Conclusions first inside each label.

## Never

- Draft without listing/reading committed changed files and the branch diff.
- Freeform section titles instead of `Why:` / `How:` / `Testing:`.
- A “Please review:” checklist.
- Diagrams inline under `How:` without `---` fences.
- Empty `---` pair when there is no diagram.
- Treat this ask as “implement the feature.”
- Invented validation or file dump with no why.

## Example (small fix — no diagram)

```text
Why: `Tesla - Lynx` site data (STST-SM-30162, STST-SM-30164) is not on the kafka producer whitelist, so it is not mirrored into the target env.

How: Same whitelist as the other Tesla sites — add both gateway IDs to kafkaProducerMirroringCriteria in `kcr-mirroring-config.yaml`.

Testing: Config review only — confirm both gateway IDs appear in the mirrored criteria list for the target env.
```

## Example (structure matters — diagram fenced)

```text
Why: An empty file list is accepted, so callers get silent empty success instead of a construction error.

How: Fail closed at the type boundary: `NonEmptyFiles` is built only via `try_from`; the shell maps that error to the API response.

---

caller → shell → NonEmptyFiles::try_from → plan

---

Testing: `cargo test planner::non_empty` — empty list fails; single file ok.
```
