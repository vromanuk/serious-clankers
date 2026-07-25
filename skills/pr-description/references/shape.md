# PR description shape

Load after you have inspected **committed** branch changes (changed file list + diff vs base). Do not draft from memory alone.

**Always the same structure.** Labels and order are fixed — do not invent headings.

```text
Why: <real need / problem in plain words>

How: <what changed and how it works>

Testing: <commands run, cases covered, or honest manual check>
```

When there is a **diagram or changed-components** sketch, put it in its **own block between `---` lines** — not mixed into the `How:` paragraph:

```text
Why: <real need / problem in plain words>

How: <what changed and how it works>

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
| `Why:` | always | Real need / problem. High-level idea OK. Not a file list. |
| `How:` | always | Mechanism in prose: what changed, where, how it works. No diagram here. |
| `---` block | when structure matters | Diagram and/or changed components only, between two `---` lines. |
| `Testing:` | always* | What was run or what to run. |
| `Risks:` / `Notes:` | when material | Compat, rollout, follow-ups. |

\*Omit `Testing:` only if the user explicitly asked for Why/How only.

- Line starts with the label exactly: `Why:`, `How:`, `Testing:`.
- One-line form: `Why: need here.`
- Multi-line form: label line, then indented or plain continuation lines.
- **Do not** use `## Summary` / `## How` / freeform titles as substitutes.

## Why

1. Restate the **real need** (first principles; strip ticket jargon if you can).  
2. High-level idea is fine — essence of the problem/change, not mechanics.  
3. Label *assumed* if the need is only inferred from the diff.

## How

- Explain the mechanism in plain words (what you did + key names).  
- **Do not** put diagrams or component maps inside this paragraph.

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

- Plain words. No corporate filler.  
- Short. Facts vs assumptions.  

## Tiny-fix shape (no diagram)

```text
Why: `Tesla - Lynx` site data (STST-SM-30162, STST-SM-30164) is not whitelisted.

How: Added both `Tesla - Lynx` gateway IDs (STST-SM-30162, STST-SM-30164) to the kafkaProducerMirroringCriteria list in `kcr-mirroring-config.yaml`.

Testing: Confirm both IDs are present in the criteria list after deploy / config apply.
```

## Structure-matters shape (with `---` block)

```text
Why: Callers get silent empty success when the planner receives an empty file list; we need a clear reject at construction.

How: `NonEmptyFiles` is built only via `try_from`; the shell maps the error to the API.

---

caller → shell → NonEmptyFiles::try_from → plan

---

Testing: `cargo test planner::non_empty` — empty list fails; single file ok.
```
