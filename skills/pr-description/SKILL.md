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
Why: <real need / problem>

How: <what changed and how it works>

Testing: <what was run or what to run>
```

When structure matters, add a **separate** diagram/components block between `---` lines (not inside `How:`):

```text
Why: <real need / problem>

How: <prose mechanism only>

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
   - Issue / ticket text only if the user gave it.
   - Uncommitted dirty files: ignore for the PR body unless the user asked to include WIP; if you skip them, do not claim they are in the PR.
2. **Analyze** — from that file list + diff: real need, mechanism, whether multiple components need a diagram, what testing evidence exists (new tests in the diff, session commands).
3. **Draft** using `references/shape.md` (load it). Labels must match exactly.
4. **Check** — grounded in committed files/diff; `Why:` → `How:` → optional `---` diagram → `Testing:`; no invented runs.
5. **Deliver** the full body (and put it in the PR if the user asked to open/update one).

## Fixed structure (mandatory)

| Label | Job |
|-------|-----|
| `Why:` | Real need / problem in plain words |
| `How:` | Prose mechanism only (no diagram here) |
| `---` … `---` | Diagram / changed components when structure matters |
| `Testing:` | Commands, cases, or honest manual check |

Optional after those when material: `Risks:` or `Notes:`.

### Rules

- Start lines with exactly `Why:`, `How:`, `Testing:`.
- **Do not** replace them with `## Summary` or freeform headings, or reorder them.
- Tiny one-liner PRs still use the same labels. No empty `---` pair.

### Why

- Restate the real need from the change (and ticket if given). Label *assumed* if only inferred from the diff.
- Not a file dump.

### How

- Mechanism from the **committed** diff: what changed and where (paths / symbols when useful).
- **No** diagram inside this paragraph.

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

- Direct. Plain words. No corporate fluff.
- Short. Conclusions first inside each label.

## Never

- Draft without listing/reading committed changed files and the branch diff.
- Freeform section titles instead of `Why:` / `How:` / `Testing:`.
- Diagrams inline under `How:` without `---` fences.
- Empty `---` pair when there is no diagram.
- Treat this ask as “implement the feature.”
- Invented validation or file dump with no why.

## Example (small fix — no diagram)

```text
Why: `Tesla - Lynx` site data (STST-SM-30162, STST-SM-30164) is not whitelisted.

How: Added both `Tesla - Lynx` gateway IDs (STST-SM-30162, STST-SM-30164) to the kafkaProducerMirroringCriteria list in `kcr-mirroring-config.yaml`.

Testing: Config review only — confirm both gateway IDs appear in the mirrored criteria list for the target env.
```

## Example (structure matters — diagram fenced)

```text
Why: Callers get silent empty success when the planner receives an empty file list; we need a clear reject at construction.

How: `NonEmptyFiles` is built only via `try_from`; the shell maps the error to the API.

---

caller → shell → NonEmptyFiles::try_from → plan

---

Testing: `cargo test planner::non_empty` — empty list fails; single file ok.
```
