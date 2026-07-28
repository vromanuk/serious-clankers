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
6. **Functions do work** — name the action (`load_profile`, `resolve_files`).  
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
4. Function name is an action when it does work?  
5. Would a better name remove a comment that only explains the name?  
6. Same style as the rest of the file?  
7. **If public/API surface:** apply § Public API naming (units in names, concrete verbs, pairs match, no double negatives, naming matches type)?  

### Flag

- Unclear public or module-level names  
- Very long names that only repeat local context  
- `helper1`, `doStuff`, `process_data` with no real meaning  
- Comment only needed because the name is opaque → rename  
- Public API smells from § Public API naming (hidden writes, bare units, vague `get`, mismatched pairs, …)  

Not a flag: normal short loop indices; domain words the project already uses.

---

## Related

| Idea | Where |
|------|--------|
| Prefer rename over a comment that only explains the name | comments stage |
| Types that force good use | architecture |
| One function per task (step names) | conventions |
