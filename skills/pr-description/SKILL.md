---
name: pr-description
spec_hash: 4917138f025b
description: >
  Draft pull-request descriptions from committed branch changes. Use when the
  user asks to write, draft, or generate a PR description, fill a PR body, or
  summarize changes for a PR. Not for full multi-lens code review or commit
  messages alone.
---

# PR description

Write a PR body a reviewer can use: why the change exists, how it works, how it was checked. Same labels, same order, every time:

```text
Why: <the missing contract and its consequence>

How: <intuition for the approach, then the mechanism>

---

<diagram or changed components — only when the change crosses components>

---

Testing: <what was run or what to run>

Notes: <optional: review tips, risks, follow-ups>
```

No other labels, no `## Summary`-style headings, no `Idea:` line.

## Workflow

1. **Read the committed changes.** Base = the branch the user named, else the default branch. Run `git merge-base HEAD <base>`, then `git log --oneline <merge-base>..HEAD`, `git diff --name-status <merge-base>...HEAD`, and read enough of `git diff <merge-base>...HEAD` to explain the change. Commit subjects alone are not enough. Ignore uncommitted edits unless the user asked to include work in progress.
2. **Fetch the ticket** if a key is in the branch, title, or user text. Use it for the need; do not paste it.
3. **Draft** with the rules below, then reread each sentence and cut what a reviewer would not miss.
4. **Deliver** the body, and put it in the PR if the user asked to open or update one.

## Why

The missing contract and what it causes, in the system's own names, so a reviewer new to the area gets the need.

- **Feature:** the value that is not available (signal, field, row) and the caller that cannot get it.
- **Fix:** open with what the code does or fails to guarantee, as the precise mechanism, then the observed impact in a sentence or two as evidence. "The TCP connect blocks until the kernel stops retrying the SYN (~2 min on Linux)", not "it gets no reply and waits". Not "In the 2026-10-19 rollout, …".
- Replace team shorthand with plain words, or define it on first use ("two replicas per shard", not "A/B").

## How

The approach first (for example, the existing pattern it extends), then the mechanism, with paths or symbols where they help. If a refactor exists only to make the change possible, say why it was needed and whether it changes behavior.

- Open with a full sentence that says what the PR adds or changes ("Adds helm values for … in …"), not a noun phrase ("PRD values for …").
- Every bullet is a full sentence a reviewer understands without the diff. No fragments joined by colons or semicolons ("12Gi pods, 2 to 6 replicas: …").
- Keep only the decisions a reviewer should judge. Leave out what code comments, values files or the ticket already carry, and side changes (image bumps, doc or link tweaks).
- If the change is one step of a rollout or migration, state the plan as its own bullet. Say when and how the new thing gets used; do not call it unused or "nobody reads it" if a later step uses it.
- Technical, plain wording; no casual phrasing.

## Testing

Short: which tests cover the change and where it was verified, with a link to the ticket or PR that holds the evidence. No command output, counts or step lists.

- For a config-only or deploy PR, point to the tests and verification of the code it deploys (for example, "covered by unit tests and verified in eng: TEI-…").
- Never a bare "N/A" or "tested". Claim only runs the session, the user, or a linked ticket or merged PR shows.

## Leave out

Anything that does not help a reviewer judge the change: file or env-var inventories, commit-by-commit lists, raw metric dumps, reproduction commands, line counts. Point to the ticket or commit instead. No "please review" checklists or filler. Do not implement anything when only the description was asked for.

## Examples

Small feature, no diagram:

```text
Why: `Tesla - Lynx` site data (STST-SM-30162, STST-SM-30164) is not on the kafka producer whitelist, so it is not mirrored into the target env.

How: Same whitelist as the other Tesla sites: add both gateway IDs to kafkaProducerMirroringCriteria in `kcr-mirroring-config.yaml`.

Testing: Config review only. Confirm both gateway IDs appear in the mirrored criteria list for the target env.
```

Deploy step of a migration (config only):

```text
How: Adds helm values for the site and device signals relays in us-smf00-prd-energy, in the same shape as eng. The new relays run next to the legacy relay and write to `telemetry-bulk-v2-prd`.
- Migration plan: new partners are served from the new bucket from the start. About 25 days later (the bucket's file retention), the existing partners move to it, and the legacy relay is turned off.
- Each pod gets 12Gi of memory, and the relays scale between 2 and 6 pods, because PRD keeps far more rows than eng.

Testing: The relay is covered by unit tests and was verified in eng against the legacy export: TEI-36123.
```

Fix with an enabling refactor:

```text
Why: `create_balanced_channel` opens connections without a connect timeout. If the TCP handshake never completes, as can happen right after the server pod restarts, the connect blocks until the kernel stops retrying the SYN (~2 min on Linux), and every request on that channel waits behind it until its own request timeout.

latest-api hit this during the last latest-data deploy. latest-data runs two replicas per shard and restarts them one at a time, and latest-api sends each read to both, so one replica being down normally goes unnoticed. This time latest-api took ~2 min to reconnect to the first replica after its restart. The second one restarted in the meantime, so for ~90 s neither was reachable and Powergate's requests failed at its 5 s deadline. TEI-36727.

How: `GrpcClientConfig` gets `connect_timeout` (default 1 s) and `create_balanced_channel` applies it, so a stuck connect fails fast and the read is served by the other replica. The field is why many files change: the struct had no constructor, so every hand-written config listed all fields. The PR adds `GrpcClientConfig::new(host, api_port)` and moves those configs to struct update syntax, with no behavior change.

Testing: `cargo test -p grpc-common`; `call_fails_within_three_seconds_when_the_server_never_accepts_the_connection` failed before the fix. ENG restart under load: ~5 s requests for 2 min before, slowest 0.69 s after (commands in TEI-36727).

Notes: The refactor commit is mechanical; check that each config keeps its non-default values.
```
