---
name: pr-description
spec_hash: 473461aa5968
description: >
  Draft pull-request titles and descriptions from committed branch changes. Use when the
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

## Title

Ticket key, a colon, then one sentence that starts with a verb and names what changes and for which service, table or component:

```text
ABC-123: Add staging DNS for the orders-api
```

No team, area or type prefix after the ticket key (`payments:`, `infra:`, `feat:`), even when older titles in the repo have one: it does not say what changes, and a reader cannot tell whether it names a team, a folder or a service.

## Workflow

1. **Read the committed changes.** Base = the branch the user named, else the default branch. Run `git merge-base HEAD <base>`, then `git log --oneline <merge-base>..HEAD`, `git diff --name-status <merge-base>...HEAD`, and read enough of `git diff <merge-base>...HEAD` to explain the change. Commit subjects alone are not enough. Ignore uncommitted edits unless the user asked to include work in progress.
2. **Fetch the ticket** if a key is in the branch, title, or user text. Use it for the need; do not paste it.
3. **Draft** with the rules below, then reread each sentence and cut what a reviewer would not miss.
4. **Deliver** the body, and put it in the PR if the user asked to open or update one.

## Why

The missing contract and what it causes, in the system's own names, so a reviewer new to the area gets the need.

- **Feature:** the value that is not available (signal, field, row) and the caller that cannot get it.
- **Fix:** open with what the code does or fails to guarantee, as the precise mechanism, then the observed impact in a sentence or two as evidence. "The TCP connect blocks until the kernel stops retrying the SYN (~2 min on Linux)", not "it gets no reply and waits". Not "In the March 3 rollout, …".
- **Step toward a larger goal** (migration, cutover, teardown): open with one sentence naming the goal, then what blocks it and why. Without it, the change has no visible purpose.
- Replace team shorthand with plain words, or define it on first use ("two replicas per shard", not "A/B").
- Write natural sentences: a subject doing something with an ordinary verb. "Only that certificate can read customer accounts", not "that certificate is the one granted"; "the certificate is defined in the old service's chart", not "the old service is what creates it".

## How

The approach first (for example, the existing pattern it extends), then the mechanism, with paths or symbols where they help. If a refactor exists only to make the change possible, say why it was needed and whether it changes behavior.

- Open with a full sentence that says what the PR adds or changes ("Adds helm values for … in …"), not a noun phrase ("Production values for …").
- Every bullet is a full sentence a reviewer understands without the diff. No fragments joined by colons or semicolons ("12Gi pods, 2 to 6 replicas: …").
- Keep only the decisions a reviewer should judge. Leave out what code comments, values files or the ticket already carry, and side changes (image bumps, doc or link tweaks).
- If the change is one step of a rollout or migration, state the plan as its own bullet. Say when and how the new thing gets used; do not call it unused or "nobody reads it" if a later step uses it.
- Technical, plain wording; no casual phrasing.

## Testing

Short: which tests cover the change and where it was verified, with a link to the ticket or PR that holds the evidence. No command output, counts or step lists.

- For a config-only or deploy PR, point to the tests and verification of the code it deploys (for example, "covered by unit tests and verified in staging: ABC-…").
- Never a bare "N/A" or "tested". Claim only runs the session, the user, or a linked ticket or merged PR shows.

## Leave out

Anything that does not help a reviewer judge the change: file or env-var inventories, commit-by-commit lists, raw metric dumps, reproduction commands, line counts. Point to the ticket or commit instead. No "please review" checklists or filler. Do not implement anything when only the description was asked for.

## Examples

Small feature, no diagram:

```text
Why: Warehouses WH-104 and WH-105 are not on the replication allowlist, so their orders are not copied to the reporting database.

How: Adds both warehouse IDs to `replicatedWarehouses` in `replication-config.yaml`, next to the existing warehouses.

Testing: Config review only. Confirm both IDs appear in the rendered allowlist.
```

Deploy step of a migration (config only):

```text
How: Adds helm values for the new export worker in production, in the same shape as staging. It runs next to the legacy exporter and writes to the `exports-v2` bucket.
- Migration plan: new customers are served from the new bucket from the start. After the bucket's 30-day retention, the existing customers move to it, and the legacy exporter is turned off.
- Each pod gets 12Gi of memory, and the worker scales between 2 and 6 pods, because production holds far more rows than staging.

Testing: The worker is covered by unit tests and was verified in staging against the legacy export: ABC-123.
```

One step of a teardown (config only):

```text
Why: We want to tear down the legacy exporter (its service, database and old bucket). Before we can remove it, the new export worker has to stop depending on it. Today the worker uses the legacy exporter's client certificate, since only that certificate can read customer accounts from the accounts API. The certificate is defined in the legacy exporter's helm chart, so removing the legacy app would delete it too.

How: Moves ownership of the certificate to the export worker, in staging and production. The worker declares the same certificate as the legacy chart (same name, common name and secret), so cert-manager keeps the existing one and nothing changes for the accounts API.

Testing: helm template for staging and production renders a certificate identical to the legacy chart's.
```

Fix with an enabling refactor:

```text
Why: `create_balanced_channel` opens connections without a connect timeout. If the TCP handshake never completes, as can happen right after the server pod restarts, the connect blocks until the kernel stops retrying the SYN (~2 min on Linux), and every request on that channel waits behind it until its own request timeout.

orders-api hit this during the last inventory-service deploy. inventory-service runs two replicas per shard and restarts them one at a time, and orders-api sends each read to both, so one replica being down normally goes unnoticed. This time orders-api took ~2 min to reconnect to the first replica after its restart. The second one restarted in the meantime, so for ~90 s neither was reachable and checkout requests failed at their 5 s deadline. ABC-456.

How: `GrpcClientConfig` gets `connect_timeout` (default 1 s) and `create_balanced_channel` applies it, so a stuck connect fails fast and the read is served by the other replica. The field is why many files change: the struct had no constructor, so every hand-written config listed all fields. The PR adds `GrpcClientConfig::new(host, api_port)` and moves those configs to struct update syntax, with no behavior change.

Testing: `cargo test -p grpc-common`; `call_fails_within_three_seconds_when_the_server_never_accepts_the_connection` failed before the fix. Staging restart under load: ~5 s requests for 2 min before, slowest 0.69 s after (commands in ABC-456).

Notes: The refactor commit is mechanical; check that each config keeps its non-default values.
```
