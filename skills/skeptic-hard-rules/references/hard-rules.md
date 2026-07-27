# Hard rules (pass/fail)

Source for **skeptic-hard-rules**. Only items that can fail without taste debates.  
Prefer encoding as lint/CI when stable.

| Id | Rule | Signal |
|----|------|--------|
| `HR-type-line-design-header` | No type-signature lines in design headers | Comment lines that are only `Type -> Type` or language signatures |
| `HR-secret-in-error` | Do not echo secrets in errors/logs | Building error strings from keys/tokens; plain secrets when a secret manager exists |
| `HR-impossible-fallback-zero` | No silent `unwrap_or(0)` / epoch on impossible calendar or invariant paths | e.g. validated date then `.unwrap_or(0)` for epoch ms |
| `HR-new-behavior-no-test` | New behavior in thinking code without a test that asserts the contract | Diff adds decision logic, no test update |
| `HR-private-import` | Do not import another component’s private path (past its public surface) | Cross-component use of private modules / non-exported paths; not merely “missing an `internal/` folder name” |
| `HR-io-in-decision-core` | No network/disk/env/clock/random inside pure decision helpers under review | `std::fs`, `Instant::now`, env, sockets inside functions documented as pure/thinking |
| `HR-prefer-battle-tested-lib` | Prefer a standard, battle-tested crate over hand-rolled code for well-solved domains (especially **date/time/calendar**, also UUID, URL, crypto primitives, compression, etc.) when the crate is already in the workspace or is the normal Rust ecosystem choice | Custom leap-year / `days_in_month` / manual `YYYY-MM-DD` validation; home-grown base64/UUID; reinventing parsers already covered by `chrono`/`time`/`uuid`/… |

### `HR-prefer-battle-tested-lib` (detail)

**Invalid:** hand-rolled calendar validation while `chrono` (or equivalent) is available or already used in the repo.

**Valid:** `NaiveDate::parse_from_str(s, "%Y-%m-%d")` (or project-standard date crate); thin wrapper only for CLI/error wording.

**Not a hit:** tiny pure helpers that are the product domain; one-liners that only format after a library parse.

**Fix:** replace with the standard crate; drop parallel calendar math.

When adding a rule: exact signal, invalid example, valid example, prefer tool enforcement later.

## Not hard rules (judgment — other stages)

| Topic | Guidance |
|-------|----------|
| **Ticket IDs in comments** | Prefer stating the **constraint**. Tickets OK for scars/critical bugs. Not on ordinary logic. |
