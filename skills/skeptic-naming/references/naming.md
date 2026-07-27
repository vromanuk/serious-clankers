# Naming

**Review question:** Did the developer pick good names for everything?  
A good name is **long enough to fully say what the item is or does**, without being **so long that it’s hard to read**.

Ideas from [Google C++ Style Guide — Naming](https://google.github.io/styleguide/cppguide.html#Naming) (Choosing Names).  
**Spelling style** (`snake_case` vs `CamelCase`): follow **this project and language** — don’t force C++ function names onto Rust.

---

## Choosing names

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

### Flag

- Unclear public or module-level names  
- Very long names that only repeat local context  
- `helper1`, `doStuff`, `process_data` with no real meaning  
- Comment only needed because the name is opaque → rename  

Not a flag: normal short loop indices; domain words the project already uses.

---

## Related

| Idea | Where |
|------|--------|
| Prefer rename over a comment that only explains the name | comments stage |
| Types that force good use | architecture |
| One function per task (step names) | conventions |
