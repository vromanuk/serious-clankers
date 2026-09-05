# PR description shape

Load after you have inspected **committed** branch changes (changed file list + diff vs base). Do not draft from memory alone.

**Always the same structure.** Labels and order are fixed — do not invent headings.

```text
Why: <why we need this — missing contract + short intuition>

How: <intuition for the approach, then the mechanism>

Testing: <commands run, cases covered, or honest manual check>
```

When there is a **diagram or changed-components** sketch, put it in its **own block between `---` lines** — not mixed into the `How:` paragraph:

```text
Why: <need + intuition>

How: <approach intuition, then mechanism>

---

<diagram and/or changed components only>

---

Testing: <commands run, cases covered, or honest manual check>
```

Optional when material (after the labels above):

```text
Risks: <rollout, compat, follow-ups>
```

or

```text
Notes: <same idea>
```

## Label rules

| Label | Required | Content |
|-------|----------|---------|
| `Why:` | always | Why we need this: named missing value, so the caller cannot fetch it. Intuition in the same paragraph. |
| `How:` | always | Intuition for the approach, then the mechanism. No diagram here. |
| `---` block | when structure matters | Diagram and/or changed components only, between two `---` lines. |
| `Testing:` | always* | What was run or what to run. |
| `Risks:` / `Notes:` | when material | Compat, rollout, follow-ups. |

\*Omit `Testing:` only if the user explicitly asked for Why/How only.

- Line starts with the label exactly: `Why:`, `How:`, `Testing:`.
- One-line form OK for tiny PRs.  
- **Do not** use `## Summary` / `## How` / freeform titles as substitutes.
- Do not add an `Idea:` line. Put that explanation under `Why:` / `How:`.

## Why (need + intuition)

`Why:` is **why we need this change**. Model first, PR-short.

1. **What is not there** — named signal / field / row in the system's names.  
2. **So the caller cannot fetch it** — which store/RPC lookup fails even though producers send it.  
3. **Who needs it** — one line + ticket if fetched. Label *assumed* if only inferred from the diff.

| Weak `Why:` | Strong `Why:` |
|-------------|---------------|
| “never saw firmware” | Wall-connector `SITE_SM_customerVersion` was not in the latest text store, so `GetLatestTextTelemetryByDin` could not return it |
| File list of what moved | Named missing value + who cannot fetch it |
| Full how-to of the patch | Leave that for `How:` |

## How (approach intuition, then mechanism)

1. **Intuition** — how we are going about it: the existing layout this extends (one ingest deployment per product; the other value type already served).  
2. **Mechanism** — ingest path, keep policy, routing. A few sentences. Paths / symbols when they help.

Not: env-var / metric / clone inventory. Not a “please review” list. Name what a component does (parquet to S3; recover by load snapshot then Kafka replay), not a slogan.

**Do not** put diagrams or component maps inside this paragraph.

## Diagram / changed components (`---` block)

When the change crosses components or boundaries:

1. Close `How:` (prose only).  
2. Blank line, then a line that is only `---`.  
3. The diagram and/or short list of changed components (ASCII arrows, boxes, names).  
4. Blank line, then a line that is only `---`.  
5. Then `Testing:`.

- That block is **only** structure (diagram / components) — not more Why/How prose.  
- Tiny single-site fix → **omit** the whole `---` block (no empty fences).

## Testing

| Good | Bad |
|------|-----|
| `cargo test planner::` — covers empty input and overflow | N/A |
| Manual: hit `/v1/x` with missing auth → 401 | “tested” |
| No automated test yet; plan: unit test for `parse_window` | “no testing required” (alone) |

## Language

- Technical names, human sentences. No corporate filler.  
- Short. Facts vs assumptions.  

## Tiny-fix shape (no diagram)

```text
Why: `Tesla - Lynx` site data (STST-SM-30162, STST-SM-30164) is not on the kafka producer whitelist, so it is not mirrored into the target env.

How: Same whitelist as the other Tesla sites — add both gateway IDs to kafkaProducerMirroringCriteria in `kcr-mirroring-config.yaml`.

Testing: Confirm both IDs are present in the criteria list after deploy / config apply.
```

## Structure-matters shape (with `---` block)

```text
Why: An empty file list is accepted, so callers get silent empty success instead of a construction error.

How: Fail closed at the type boundary: `NonEmptyFiles` is built only via `try_from`; the shell maps that error to the API.

---

caller → shell → NonEmptyFiles::try_from → plan

---

Testing: `cargo test planner::non_empty` — empty list fails; single file ok.
```
