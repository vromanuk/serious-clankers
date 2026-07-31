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
| `HR-api-get-mutates` | Public **mutating** operations must not look like pure reads | `get_*` / `fetch_*` / HTTP `GET` that create/update/delete or have real side effects |
| `HR-api-bare-quantity` | Public quantities must name the **unit or standard** (or carry currency) | Public field/param `duration` / `date` / `amount` / `price` as bare number/string with no unit/standard in name or structure |
| `HR-api-bool-status` | Public booleans must name a **state**, not vague `status` | Public `status: bool` (or `status: true/false` in wire JSON) for a yes/no flag |
| `HR-api-double-negation` | Public flags must not force **double-negation** reading | Public bool `dont_*` / `do_not_*` / `no_*` where callers set `false` to allow, or pair of `no_x` flags instead of `has_x` |
| `HR-api-name-type-mismatch` | Public name must match **what the value is** | Field named `recipe` but only an id; named `*_id` but holds a full object; singular collection path that returns a list with no list/marker |

### `HR-prefer-battle-tested-lib` (detail)

**Invalid:** hand-rolled calendar validation while `chrono` (or equivalent) is available or already used in the repo.

**Valid:** `NaiveDate::parse_from_str(s, "%Y-%m-%d")` (or project-standard date crate); thin wrapper only for CLI/error wording.

**Not a hit:** tiny pure helpers that are the product domain; one-liners that only format after a library parse.

**Fix:** replace with the standard crate; drop parallel calendar math.

### API / interface hard rules (detail)

Scope: **public** surfaces only — `pub` API, HTTP/JSON wire fields, SDK methods, component roots. Private helpers are naming-stage judgment, not these HRs.  
Source ideas: [The API Book — Describing Interfaces](https://twirl.github.io/The-API-Book/API.en.html#api-design-describing-interfaces-para-3).

#### `HR-api-get-mutates`

**Invalid:** `get_or_create_user` that writes; HTTP `GET /orders` that creates an order; `fetch_and_delete`.  
**Valid:** `create_order`, `cancel_order`, `POST`/`PUT`/`DELETE` for writes; true reads named `get_*`.  
**Not a hit:** internal private fn named `get_x` that mutates local cache only and is not public API.  
**Fix:** rename to a modifying verb; change HTTP method if this is a web API.

#### `HR-api-bare-quantity`

**Invalid:** public `"duration": 5000`, `"date": "11/12/2020"`, `"amount": 10` with no unit/currency/standard in name or sibling fields.  
**Valid:** `duration_ms`, `iso_date` / `created_at`, `amount` + `currency`, `{ "unit": "ms", "value": 5000 }`.  
**Not a hit:** domain type `Duration` / `chrono::Duration` where the type itself is the unit; private locals.  
**Fix:** put unit or standard in the name, or use a typed structure with unit/currency.

#### `HR-api-bool-status`

**Invalid:** public wire/API field `status: bool` meaning finished/open/ready.  
**Valid:** `is_finished`, `is_ready`, `open_now`; or `status` as an **enum**/string of named states (not a bare bool).  
**Not a hit:** HTTP status codes; enums named `Status`.  
**Fix:** rename bool to a qualitative state, or use a multi-value status type.

#### `HR-api-double-negation`

**Invalid:** public `"dont_call_me": false`, `"no_beans": false` + `"no_cup": false` as the way to say “allowed / has resources”.  
**Valid:** `prohibit_calling: true`, `has_beans` + `has_cup`.  
**Not a hit:** domain terms that are naturally negative once (`disabled: true` meaning off is OK if positive form would be worse and call sites stay simple).  
**Fix:** name the positive state callers combine with `&&`.

#### `HR-api-name-type-mismatch`

**Invalid:** public field `recipe` that is only a UUID; `recipe_id` that embeds a full `Recipe`; `GET /news` that returns an array with no list cue in the name when singular/plural is ambiguous.  
**Valid:** `recipe` → object; `recipe_id` → id; lists plural or `…_list`.  
**Not a hit:** newtypes that wrap an id but are named for the domain type by project convention (document once).  
**Fix:** align name with payload kind (object vs id vs list).

When adding a rule: exact signal, invalid example, valid example, prefer tool enforcement later.

## Not hard rules (judgment — other stages)

| Topic | Guidance |
|-------|----------|
| **Ticket IDs in comments** | Prefer stating the **constraint**. Tickets OK for scars/critical bugs. Not on ordinary logic. |
| **Vague but not false public names** | e.g. `get_time()` when several times exist; short `str` — **naming** stage (API Book “don’t spare letters”) |
| **Function verb / bool predicate form** | action = verb phrase; bool = `is`/`has`/`can` — **naming** stage (`naming.md` § Function naming), not automatic HR |
| **Shallow public surface / missing use-case orchestration** | many exported step-helpers or handler-owned workflows — **architecture** (`components.md`, `design-components`), not automatic HR |
| **Formal DDD/Cosmic stack by default** | repo/UoW/aggregate types without pain — judgment / design-components “good enough”, not HR |
| **Mismatched pair words** | e.g. `begin`/`stop` — **naming** stage unless it also breaks a hard rule above |
| **Optional bool default confusion** | prefer clear types — **naming** / architecture, not automatic HR |
