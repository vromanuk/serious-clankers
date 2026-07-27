# Comments — Google ideas (any language)

From [Google C++ Style Guide — Comments](https://google.github.io/styleguide/cppguide.html#Comments).  
Same ideas work in Rust, Go, Java, TypeScript, etc. Use the project’s comment syntax. Skip pure C++ header rules unless you are in C++.

**This skill:** `comments.md`, `design-headers.md`.  
**Naming:** `skeptic-naming`. **Composition / style:** `skeptic-conventions`.

---

## General ideas

### Why comment at all

Comments help the **next person** (often you later). Be generous when something is not obvious.

**Best code explains itself.** Clear names beat obscure names that need a comment. Prefer rename over a comment that only supplies the missing verb.

### Two places, two jobs

| Place | Job |
|-------|-----|
| **Public type / function / module** | How to **use** it: purpose, inputs/outputs, rules, failure, who owns args, threading if it matters. **Not** a full how-it-works essay. |
| **Inside the body** | How it **works** when tricky: odd steps, why this approach, lock/order hazards. **Do not** paste the public blurb again. |

Rust: `///` / `//!` on the item vs `//` inside the body.

### File / module

If a file or module holds related public pieces, a short top note can say what belongs here and what does **not**. Detail lives on each type/function, not a novel at the top.

Rust: module `//!` for **purpose**, key rules, **entrypoints** (what to call first), related modules, and a short multi-step flow when useful — see `comments.md` § Module docs. Skip empty filler.

### Types / classes / structs

Non-obvious types get a comment: **what it is for, how to use it**, important constraints (e.g. threading). A small usage snippet is fine when it helps.

Skip “this class holds X” when the name and fields already say that.

### Functions

**On the public declaration / signature:**

- What it does and how to use it (when non-obvious).  
- Prefer a short action phrase when useful (“Opens the file”, “Returns an iterator at…”).  
- Inputs/outputs; empty/null rules; whether args are kept after the call; overwrite vs append; costly use if it matters.  
- **Skip** when simple and obvious (trivial getters).  
- Overrides: only what’s special about *this* one — don’t restate the base.

**Inside the body:**

- Tricky *how* and *why* (odd tricks, step overview, why not the obvious alternative).  
- Do **not** repeat the full public comment.

**Constructors / setup:** say who owns args and non-obvious cleanup; skip “creates the object.”

This pack also: non-trivial functions → **purpose + given/expected** (no type-signature line) — see `design-headers.md`.

### Variables / fields

Names should usually be enough. Comment when:

- **rules** or links between fields the type doesn’t show,  
- **special values** (`-1` = unknown size, empty list vs missing, …),  
- **globals** — what they are, and why global if unclear.

Skip “// event count” on `num_events`.

### Inside the body

Comment **tricky or important** parts — before a hard block when needed.

**Call arguments** when meaning is unclear — prefer, in order:

1. Named constants if the same magic number is shared.  
2. Enum / typed options instead of bare true/false flags.  
3. One options struct for many knobs.  
4. Named locals instead of huge nested expressions.  
5. **Last resort:** a short comment on a literal at the call.

### Writing quality

Treat longer comments like normal prose: spelling, grammar, readable sentences. End-of-line nits can be shorter; stay consistent.

### TODO

For temporary, short-term, or “good enough but not perfect” code:

- `TODO` in all caps.  
- Who or what owns the context (bug id, design doc, person).  
- If time-based: a **specific** date or event (“remove when clients support X”), not “someday.”

Prefer stating the **constraint** over ticket-only lines; tickets OK as scars when needed (`comments.md`).

---

## Do not state the obvious

Do **not** literally describe what the code does, unless the behavior is non-obvious to a reader who already knows the language well.

Instead:

- write a **higher-level** comment that says **why** (or the non-obvious intent), or  
- **rename / restructure** so the code is self-describing and needs no comment.

### Bad — restates the code

```cpp
// Find the element in the vector.  <-- Bad: obvious!
if (std::find(v.begin(), v.end(), element) != v.end()) {
  Process(element);
}
```

```rust
// Loop over files and get the name.  <-- Bad: obvious!
for f in files {
    let name = f.name();
    ...
}
```

### Better — higher-level why / intent

```cpp
// Process "element" unless it was already processed.
if (std::find(v.begin(), v.end(), element) != v.end()) {
  Process(element);
}
```

```rust
// Skip files already in the index; only new names are ingested.
for f in files {
    let name = f.name();
    ...
}
```

### Best — clear code (no comment needed)

```cpp
if (!IsAlreadyProcessed(element)) {
  Process(element);
}
```

```rust
for f in files.iter().filter(|f| !index.contains(f.name())) {
    ingest(f);
}
```

If a good name or extract makes the “why” obvious, **delete** the comment. Do not keep a comment that only paraphrases the next line.

---

## Map onto this pack

| Idea | Where |
|------|--------|
| Clear names first | Naming stage |
| Public docs vs body comments | `comments.md`; this file |
| Module purpose | `comments.md` |
| Purpose + examples | `design-headers.md` |
| Hazards, ownership, scars | `comments.md` examples |
| Don’t restate the next line | This file § Do not state the obvious |
| Prefer real types over bool flags at calls | Architecture; this file |

---

## Reviewing comments (checklist)

Use this when judging comments on a diff:

1. **Clear English?** Normal readable writing.  
2. **Actually needed?** If the code already says it, delete or simplify the code.  
3. **Why, not what?** Good comments explain **why** the code exists or a decision. They should not restate **what** the next lines do.  
4. **Unclear code → simplify first.** Don’t paper over hard code with a long comment.  
5. **Exceptions (short *what* is OK):** regex, dense bit/math, hard algorithms, odd protocols. Still prefer a clear name + one line of *why* when both fit.  
6. **Only what code can’t say:** reasoning, product rule, scar, ownership, “do not change to X” — not a paraphrase of the syntax.

### Flag

- Comment only restates the next line (**what**, not **why**)  
- Comment only needed because the code is overcomplicated → simplify, don’t add more docs  
- Body repeats the full public blurb  
- Field comment is “this is a count” with no special rule  
- Vague name that only works because of a comment → rename  
- TODO with no owner/context and no end condition when it is time-based  
- Missing *why* on truly non-obvious code  
- Comment text that is hard to read  

Not a flag: short ownership, special values, concurrency, decision rationale, or “why not the obvious alternative”; brief *what* on regex/hard algorithms when it really helps.
