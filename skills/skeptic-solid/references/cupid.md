# CUPID: properties to move towards

Reference for Dan North's CUPID properties, used in this review as the counterweight to SOLID: a direction to move in, not rules to comply with.

North: "Principles are like rules: you are either compliant or you are not." Properties instead "define a goal or centre to move towards. Your code is only closer to or further from the centre, and there is always a clear direction of travel." And: "everything is a trade-off so you should always consider your context."

Martin's reply is the other half: "The best way to make a complicated mess is to tell everyone to 'just be simple' and give them no further guidance." So a finding names both the property and the concrete cost — never just an acronym, and never just "keep it simple".

---

## 1. The five properties

### 1.1 Composable — "plays well with others"

- **Small surface area.** A narrow, opinionated API has less to learn and less to go wrong — but too narrow, and "the right combination" becomes tacit knowledge. "There is a sweet spot of 'just right' cohesion between fragmented and bloated."
- **Intention-revealing.** "I can easily find your component, and just as quickly decide whether it is the thing I need or not."
- **Minimal dependencies.** Each dependency adds a library, often a specific version, and a chance of incompatibility.

**Flag:** `pub` where `pub(crate)` would do; callers that must call several functions in an undocumented order for one common task; generic names (`Manager`, `Helper`, `utils`) that hide the purpose; a library pulling in a heavy dependency (or its default features) for a small use.

### 1.2 Unix philosophy — "does one thing well"

"'doing one thing well' is an outside-in perspective; it is the property of having a specific, well-defined, and comprehensive purpose. SRP is an inside-out perspective: it is about the organization of code."

**Flag:** a component whose purpose a caller cannot state in one sentence, or that does part of a job and leaves callers to finish it.

### 1.3 Predictable — "does what you expect"

"Code should do what it looks like it does, consistently and reliably, with no unpleasant surprises." Predictability is "a generalization of testability":

- **Behaves as expected** — intent obvious from structure and naming; easy to test.
- **Deterministic** — "Software should do the same thing every time"; limits and edge cases are obvious.
- **Observable** — internal state can be inferred from outputs, which "is only possible when we design it in."

**Flag:** names that hide side effects (`get_*` that writes); behavior that depends on hidden state, the clock, or randomness read internally; unstated limits on memory, queues, or time. Depth on telemetry is the observability stage's.

### 1.4 Idiomatic — "feels natural"

"Code should conform to the idioms of the language." Where the language allows several styles, the team decides what good looks like and writes it down.

**Flag:** Java-style `IFoo` traits and getters, sentinel values instead of `Result`/`Option`, a different style from the rest of the codebase for the same task. Rust style details are the conventions stage's.

### 1.5 Domain-based — "models the problem domain in language and structure"

- **Language:** "defining a Surname type will be more intention-revealing" than a bare string; a casual observer should not be able to tell whether people are discussing the code or the domain.
- **Structure:** directory and module layout mirrors the domain; "grouping them by type should not form the primary structure."
- **Boundaries:** module boundaries become domain boundaries.

**Flag:** money as `f64`; identifiers of different kinds all `String` or `u64`; several `bool` parameters where an enum would name the meaning; top-level `models/`, `handlers/`, `services/` where one domain change touches all of them. Naming depth is the naming stage's; layout depth is the architecture stage's.

---

## 2. North's critique, as checks on SOLID findings

Use these to test whether a SOLID finding is real:

| SOLID finding | North's objection | Check before reporting |
|---|---|---|
| "Split this — single responsibility" | "What is one thing anyway?"; splits can be "a premature segregation with negative consequences" | Name the different reasons to change, and confirm the parts do not change together |
| "Make this open for extension" | "All code is cost"; simple code is easy to change directly | Does a real, recurring kind of change justify the extension point? |
| "Substitution is broken" | Agrees: it is "the Principle of Least Surprise applied to code substitution" | Report it — this one rarely over-applies |
| "Segregate this interface" | It is a pattern for escaping a god object, not a principle | Could the type itself be split instead? |
| "Invert this dependency" | Most dependencies "aren't options, they are just the way we are going to do it this time" | Is there more than one real way to provide it, or a boundary worth wiring separately? |

Martin and North agree on the goal — simple code that is easy to change. Martin: "Simple code is both open and closed." North: "Write simple code."

---

## Sources

- CUPID: for joyful coding (Dan North) — https://dannorth.net/blog/cupid-for-joyful-coding/
- CUPID: the back story (Dan North) — https://dannorth.net/blog/cupid-the-back-story/
- https://cupid.dev/ — CUPID
- Solid Relevance (Robert C. Martin, 2020) — https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html
