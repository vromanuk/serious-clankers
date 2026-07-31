# Naming

**Review question:** Did the developer pick good names for everything?  
A good name is **long enough to fully say what the item is or does**, without being **so long that it’s hard to read**.

**General names:** [Google C++ Style Guide — Naming](https://google.github.io/styleguide/cppguide.html#Naming) (Choosing Names).  
**Public / API surface names (stronger):** [The API Book — Describing Interfaces](https://twirl.github.io/The-API-Book/API.en.html#api-design-describing-interfaces-para-3) (Sergey Konstantinov).  

**Spelling style** (`snake_case` vs `CamelCase`): follow **this project and language** — don’t force another language’s style.

Rules are generalizations — don’t apply them blindly if they make an unusable API. Be **consistent** with whatever scheme you choose.

---

## Choosing names (any scope)

1. **Purpose first** — a new reader should get the idea from the name.  
2. **Don’t shorten public names just to save space** — clarity wins for anything others call.  
3. **How public it is**  
   - Public / shared → fuller name.  
   - Local in a tiny loop → short is fine when the context is obvious.  
4. **Don’t repeat what’s already clear** — e.g. `user.id` not `user.user_id` if the type is already `User`.  
5. **Abbreviations**  
   - Avoid ones outsiders won’t know, and don’t drop letters (`cstmr`).  
   - Fine: common ones (`i` for a loop, clear domain `id`).  
   - Prefer whole words when unsure.  
6. **Functions do work** — name the action (verb or verb phrase). See **Function naming** below.  
   **Types and values are things** — use nouns.  
7. **Plain words** — no made-up pattern nicknames as names.  
8. Most names are ordinary. Only rare, widely reused types need special shared vocabulary and docs.

### Too short / unclear

```text
proc(x)           // does what?
tmp2, data, obj   // on something others use
cstmr_id          // letters deleted
```

### Too long for a small job

```text
// Inside a small count over foos:
total_number_of_foo_errors_in_current_batch  // too much for a loop counter
// Prefer: n or error_count
```

### About right

```text
count_foo_errors(foos)
table_name
max_allowed_connections
is_already_processed   // or IsAlreadyProcessed — match the language
```

---

## Function naming (stronger)

Portable rules for **functions and methods** (any language). Spelling style still follows the project (`snake_case` vs `camelCase`).

Aligned with widespread guides: [.NET Framework Design Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/names-of-type-members) (methods = verbs; bools affirmative, often `Is`/`Can`/`Has`); [Google Java Style](https://google.github.io/styleguide/javaguide.html) (methods = verbs / verb phrases); Oracle Java conventions (methods are verbs); common Python practice for predicates (`is_` / `has_` — not a PEP 8 mandate, but the usual readable form); Rust std style (`is_empty`, `is_some`, action methods as verbs).

### 1. Actions start with a verb

A function that **does work** (computes, loads, writes, transforms, sends, cancels, …) is named with a **verb or verb phrase**. The call site should read as an action.

| Bad | Why | Better |
|-----|-----|--------|
| `profile_credentials()` | noun phrase — is this a value or work? | `load_profile_credentials()` |
| `user_stats` as a function | looks like a field | `calculate_user_stats()` / `fetch_user_stats()` |
| `file_list` | thing, not action | `list_files()` |
| `notification` | what does it do? | `send_notification()` |

**Prefer concrete verbs** over amoeba words alone: not bare `process`, `handle`, `do`, `manage`, `run` without saying *what*.

| Bad | Better |
|-----|--------|
| `process(data)` | `parse_invoice_lines(data)` / `normalize_timestamps(data)` |
| `handle(event)` | `apply_payment_event(event)` / `route_webhook(event)` |
| `do_stuff()` | name the real job |

**Types / values stay nouns.** If you need a noun at the call site, that is usually a type, field, or (in languages that have them) a property — not a free function that runs work.

### 2. Boolean returns are predicates (`is` / `has` / `can` / …)

A function or method that returns a **boolean** (or is used only as a yes/no question) must read as a **yes/no question** at the call site.

**Default prefixes** (pick the one that matches the meaning):

| Prefix | Meaning | Examples |
|--------|---------|----------|
| **`is_`** | state, identity, condition | `is_ready`, `is_empty`, `is_valid`, `is_authenticated` |
| **`has_`** | possession, presence, inclusion | `has_permission`, `has_children`, `has_expired_token` |
| **`can_`** | capability, allowance, feasibility | `can_retry`, `can_seek`, `can_connect` |

Also fine when they fit better: **`should_`** (policy / recommendation), **`are_`** / **`were_`** (plural subject), **`needs_`** (requirement). Same idea: **affirmative question**, not a vague noun.

| Bad | Why | Better |
|-----|-----|--------|
| `ready(user)` | not clearly a bool question | `is_ready(user)` |
| `check_valid(x)` | “check” is vague; often returns more than bool | `is_valid(x)` |
| `permission(user)` | thing, not question | `has_permission(user, …)` |
| `status()` → bool | vague (and public `status: bool` is a hard rule) | `is_finished()` / `is_open()` |
| `not_empty(xs)` | negative form | `is_empty(xs)` and invert at call site, or `has_items(xs)` |

**Rules of thumb**

- Prefer **positive** names (`is_enabled`, not `is_not_disabled`). Callers write `if !is_enabled` when they need the other side.  
- The name should make `if name(...)` / `if name` read as English: `if is_ready(job)`, `if has_capacity(queue)`, `if can_retry(err)`.  
- Same prefixes for **bool fields / variables** when the name stands alone (`is_active`, not bare `active` on a public or non-obvious surface). Local `ok` / `found` in a tiny block can stay short when the type and use are obvious.

**Not the same as action functions.** `validate_config` that returns `Result` is an **action** (verb). `is_valid_config` that returns `bool` is a **predicate**. Don’t mix: a `check_*` that returns `bool` is usually weaker than `is_*` / `has_*` / `can_*`.

### 3. Narrow exceptions (do not use these to dodge the rule)

- **Language / framework protocol names** you must implement (`fmt::Display`, `Iterator::next`, serialization hooks).  
- **Constructors / conversions** by project idiom (`new`, `from_str`, `of`, `default`).  
- **Operators** and symbol traits.  
- **Project-local getters** that already follow a clear house style (e.g. Go often drops `Get` on pure accessors — match **this** codebase, don’t invent a second style).  
- Returning **optional / result types**, not bool — name the action or the value (`find_user`, `load_config`), not a fake `is_*`.

If none of those apply and the name is still a bare noun or a non-predicate bool, **rename**.

### 4. Design and comments

If a `///` exists only to supply the missing verb or the missing “is this a bool?”, **rename** — don’t paper over the name. See write-code § Names carry the action and the comments stage.

---

## Public API naming (stronger bar)

For **public surfaces** — HTTP/JSON fields, SDK methods, crate/`pub` APIs, component roots — apply a **higher** standard. Reading call sites should make sense **without** the docs.

### 1. Explicit is better than implicit

The name should show **what happens** and any **side effects**.

| Bad | Why | Better |
|-----|-----|--------|
| `order.canceled = true` | Looks like a field set; hides cancel side effects | `order.cancel()` |
| `orders.get_stats()` | Sounds free; may scan all history / cost money | `orders.calculate_aggregated_stats({ begin, end })` |

- **Modifying** operations must look modifying — not `get_*` / not HTTP `GET` for writes.  
- If the API mixes sync and async, names (or a clear convention) must show which is which.  
- Expensive work: put that in the name or require explicit bounds (don’t default to “entire world”).

### 2. Name the standard / unit (don’t leave it guessed)

Humanity doesn’t agree on date formats, units, or coordinates. **Say which standard or unit you use.**

| Bad | Better |
|-----|--------|
| `"date": "11/12/2020"` | `"iso_date": "2020-11-12"` |
| `"duration": 5000` | `"duration_ms": 5000` or `"iso_duration": "PT5S"` or `{ "unit": "ms", "value": 5000 }` |
| Money as bare number | Always with **currency** |

Dates without a `Date` type (e.g. JSON): mark them — `created_at`, `occurred_at`, `…_date`.

### 3. Concrete names — not vague verbs alone

Avoid amoeba words alone: `get`, `apply`, `make`, `process`, `handle`, `do`.

| Bad | Better |
|-----|--------|
| `user.get()` | `user.get_id()` (or `get_profile`, whatever it actually returns) |
| `process(data)` | name the real job |

### 4. Don’t spare letters on public APIs

In the 21st century, don’t cryptic-abbreviate public names.

| Bad | Better |
|-----|--------|
| `order.get_time()` | `order.get_estimated_delivery_time()` (which time?) |
| `strpbrk(str1, str2)` | `str_search_for_characters(str, lookup_character_set)` |

Shortening `string` → `str` rarely helps. Shrinking field names to save bandwidth is usually pointless once the wire compresses.

### 5. Naming implies typing

| Name | Expected meaning |
|------|------------------|
| `recipe` | a `Recipe` (or full recipe object) |
| `recipe_id` | an id of a recipe, not the whole object |
| Arrays / lists | **plural** or collective: `objects`, `children`, `news_list` — not ambiguous `GET /news` |
| Booleans | a **state**, not a vague “status”: `is_ready`, `is_finished`, `open_now` — not `"status": true` |

If a word is ambiguous, add a prefix/suffix:

| Bad | Better |
|-----|--------|
| `GET …/functions` (built-ins? code? “functioning”?) | `…/builtin_functions_list` |

### 6. Matching pairs use matching words

| Bad | Better |
|-----|--------|
| `begin_transition` / `stop_transition` | `begin`/`end` **or** `start`/`stop` |

Same family of methods: same argument order, same naming style, same “first vs all” meaning made obvious in the name.

### 7. Avoid double negation

| Bad | Better |
|-----|--------|
| `"dont_call_me": false` | `"prohibit_calling": true` or a positive flag |
| `"no_beans": false` + `"no_cup": false` | `"has_beans"` + `"has_cup"` so callers use simple `&&` |

Prefer flags people can combine without De Morgan mistakes.

### 8. Optional fields with non-default meaning

If a new optional bool defaults to something non-obvious, **don’t** rely on “missing vs false” confusion without a clear type/API (three-state, enum, or explicit default in docs and name). Prefer APIs where “not set” and “set to false” are not easy to mix up.

### Public API — quick flags

Soft (this stage): vague verbs, short public abbreviations, mismatched pair words (`begin`/`stop`), ambiguous `get_time()`.

**Hard rules** (stage hard-rules — fail with id): see `skeptic-hard-rules/references/hard-rules.md`

| Id | One line |
|----|----------|
| `HR-api-get-mutates` | Mutating public op must not look like a read (`get_*` / HTTP GET that writes) |
| `HR-api-bare-quantity` | Public duration/date/money needs unit or standard in the name/shape |
| `HR-api-bool-status` | Public bool must not be bare `status` |
| `HR-api-double-negation` | Public flags must not force double-negation reading |
| `HR-api-name-type-mismatch` | Name must match payload kind (object vs id vs list) |

---

## Spelling style (project wins)

| Language (typical) | Types | Functions | Locals / fields |
|--------------------|-------|-----------|-----------------|
| Rust | `CamelCase` | `snake_case` | `snake_case` |
| Go | export rules | export rules | short / camel |
| Java / Kotlin | `CamelCase` | `camelCase` | `camelCase` |
| C++ (Google) | `PascalCase` | `PascalCase` (getters often snake) | `snake_case` |

When reviewing: **match the file you’re in**. Care more about “fits neighbors” than “matches Google C++.”

Google-only shapes (`kConstantName`, trailing `_` on class fields, `MYPROJECT_MACRO`) — only if this codebase already uses them.

---

## Review checklist

1. New/changed names: clear to a stranger?  
2. Public too short? Local too long?  
3. Odd abbreviations or dropped letters?  
4. **Function that does work:** verb / verb phrase (not a bare noun)? Concrete verb (not amoeba `process` / `handle` alone)?  
5. **Function / field that is a bool:** affirmative predicate (`is_` / `has_` / `can_` / `should_` / …)? Reads as a yes/no at the call site?  
6. Would a better name remove a comment that only explains the name?  
7. Same style as the rest of the file?  
8. **If public/API surface:** apply § Public API naming (units in names, concrete verbs, pairs match, no double negatives, naming matches type)?  

### Flag

- Unclear public or module-level names  
- Very long names that only repeat local context  
- `helper1`, `doStuff`, `process_data` with no real meaning  
- Action named as a noun (`profile_credentials()`, `user_list()` as work)  
- Bool without a predicate form (`ready()`, `check_valid()`, bare `status` as bool)  
- Comment only needed because the name is opaque → rename  
- Public API smells from § Public API naming (hidden writes, bare units, vague `get`, mismatched pairs, …)  

Not a flag: normal short loop indices; domain words the project already uses; protocol/constructor names from § Function naming exceptions.

---

## Related

| Idea | Where |
|------|--------|
| Prefer rename over a comment that only explains the name | comments stage |
| Short form when writing code (verb + bool prefixes) | conventions → `write-code.md` § Names carry the action |
| Types that force good use | architecture |
| One function per task (step names) | conventions |
| Public bare `status: bool` / double-negation flags | hard-rules (`HR-api-bool-status`, `HR-api-double-negation`) |
