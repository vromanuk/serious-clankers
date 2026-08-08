# Flashcard examples and source rules

**Load when drafting or reviewing cards.** Pair with `guide.md` (craft) and `html-deck.md` (HTML/copy/bold).

This file has two jobs:

1. **Reference rules** from SuperMemo, Anki practice, and related advice — adapted to this pack.  
2. **Worked examples** in the style of real high-quality decks (Rust smart pointers, DDD/events-first, systems) — good cards, bad cards, and what principle each illustrates.

---

## A. SuperMemo: Twenty rules (map to this pack)

Source: Piotr Wozniak, [*Effective learning: Twenty rules of formulating knowledge*](https://www.supermemo.com/en/blog/twenty-rules-of-formulating-knowledge) (1999, updated).  
Use spaced repetition; formulation quality multiplies retention speed.

| # | SuperMemo rule (short) | How we apply it |
|---|------------------------|-----------------|
| 1 | **Do not learn if you do not understand** | No cards until need + mechanism are clear. Prefer `explain-topic` first. |
| 2 | **Learn before you memorize** | Build the whole picture (chapter / model) before atomizing into cards. |
| 3 | **Build upon the basics** | Early deck cards = foundations; edge cases later. |
| 4 | **Minimum information principle** | **One idea per card.** SuperMemo wants tiny answers; we allow **moderate** answers when the idea is a design tradeoff — but still **one** idea (not a whole chapter). |
| 5 | **Cloze deletion** | Good for lists/steps; for mechanisms prefer full Q/A (contrast, when, why). |
| 6 | **Use imagery** | Prefer structural analogies that encode a constraint (Rc = TV until last person leaves). Optional in HTML: simple static figure — not required. |
| 7 | **Mnemonics** | Rare for systems; only when pure recall of a sequence. |
| 8 | **Graphic deletion** | Geography/anatomy; rarely primary for code cards. |
| 9 | **Avoid sets** | Don’t ask “list all Rc rules” as one card; split or order the list. |
| 10 | **Avoid raw enumerations** | Prefer “when to use” / ordered process / overlapping cloze if you must. |
| 11 | **Combat interference** | Near twins need sharp contrast fronts (Box vs Rc, not two soft “shared ownership” cards). |
| 12 | **Optimize wording** | Short front; scannable back; bold load-bearing words. |
| 13 | **Refer to other memories** | Link to prior cards (after you know Box, ask why Box fails for two list heads). |
| 14 | **Personalize / examples** | Domain-true: HTTP concurrency, orders, graph nodes — not random kitchen scenes for systems. |
| 15 | **Emotional / vivid cues** | Optional; don’t invent fake drama. A real production panic message can stick. |
| 16 | **Context cues** | Theme tags on the HTML deck (`contrast`, `rust`, `ddd`) so similar terms stay disambiguated. |
| 17 | **Useful redundancy** | Same fact from two angles (definition spine + when-to-use + contrast) is OK. |
| 18 | **Provide sources** | Deck header or card footer: book/talk/chapter when non-obvious. |
| 19 | **Date / version stamp** | Volatile APIs: note version if it matters. |
| 20 | **Prioritize** | Card the 20% that carries judgment; skip trivia. |

### SuperMemo Dead Sea lesson (classic)

**Ill-formed (one mega-card):** “What are the characteristics of the Dead Sea?” → paragraph of location, depth, salt, length, …

**Well-formed:** many short cards (where? lowest point? why float? …).

**Our systems adaptation:** do **not** put “everything about RefCell” on one card. Do put **one** contrast (RefCell vs Box: when rules are checked) with enough **why** that the contrast sticks — that is still one idea.

---

## B. Other internet principles (Anki / SRS communities)

| Principle | Source flavor | Pack use |
|-----------|---------------|----------|
| **Atomic cards** | Anki “atomic”, precise questions | One *questionable* idea; split independent guarantees |
| **Understand first, then Anki** | Med-school Anki guides | Same as SuperMemo 1–2 |
| **Foundations / 80–20 first** | High-yield decks | Basics and decision rules before edge lore |
| **Precise cue** | “Rules for precise Anki cards” | Front asks exactly one thing |
| **Hard to grade → split** | Effective SRS essays | If “half right” is common, two cards |
| **Holistic before isolated facts** | “Holistic then Anki” style posts | Model pass before cardization |
| **Process cards carefully** | Process/Anki writeups | Prefer ordered steps or “first step” principle, not 15-item dumps |

**Conflict with pure minimum-info culture:** pure SuperMemo wants ultra-short answers. This pack’s learner wants **intuition and why**. Resolution: **one idea**, answer **moderate and scannable** (lead + bullets + bold), not a slogan and not a chapter.

---

## C. Patterns from strong personal-style decks

What works in decks like **Rust smart pointers**, **DDD / events-first**, **systems design**:

| Pattern | Example front | Why it works |
|---------|---------------|--------------|
| Contrast | Box vs Cell | Forces decision, not synonym soup |
| Three-way matrix | Rc vs Box vs RefCell | Interference killers + judgment |
| When to use | When to use Box? | Turns knowledge into action |
| Structural analogy | Rc as family-room TV | Encodes “last one turns it off” |
| Elaborate code | Recursive List + two heads with Box | Compiler story becomes the model |
| Process / order | First step designing a microservice | Captures *how to think*, not tools |
| Why not the default | Problem with noun-first DDD | First principles of design order |
| Unit of consistency | What is an aggregate for? | Business + failure atomicity |

---

## D. Worked examples (good / bad)

Use these as **templates** when drafting. Bold = important on HTML page and as `**…**` in copy payload.

---

### D1. Contrast (from smart-pointer practice)

**Bad**

```text
Q: What is Box?
A: A smart pointer that puts data on the heap.
```

**Good**

```text
Q: **Box** vs **Cell** — when each?

A: **Box<T>** is for **single ownership** of heap data (or a known-size handle when the real type is recursive/unsized).

**Cell<T>** is **interior mutability** for **Copy** values: you can change the inside through **&T** when you need simple get/set, not shared ownership.

- Box: “where does this live / how is size known?”
- Cell: “I have &self but need to tweak a Copy field.”
```

**Principles:** SuperMemo 11 (interference), 12 (wording), 14 (sharp use cases); pack contrast + when.

---

### D2. When to use (Box)

**Bad**

```text
Q: Box
A: Heap allocation.
```

**Good**

```text
Q: When do you reach for **Box<T>**?

A: When you need a **heap** allocation with a **fixed-size** handle and **single ownership**:

- Type size **unknown at compile time** but the context needs a fixed size (e.g. recursive types).
- **Large** value: move ownership **without** copying the payload.
- Own a value only as “something that implements a trait” (**trait object**), not a concrete type.

Not: multi-owner graphs (that is **Rc**), or mutate through &T for non-Copy (often **RefCell**).
```

**Principles:** judgment card; minimum *ideas* as bullets under one “when Box” idea.

---

### D3. First principles / recursive type

**Bad**

```text
Q: Recursive types in Rust
A: Use Box.
```

**Good**

```text
Q: Why does a **recursive** enum need **Box** (or another indirection)?

A: Rust must know **size at compile time**. A type that contains itself could nest forever, so the size is not finite.

A **Box<T>** is a **pointer**: its size is always known. Putting **Box** on the recursive field makes the enum a fixed-size header + pointer to the next node.

Need: express “list of lists of …” in one type.  
Constraint: known size.  
Mechanism: indirection with known pointer width.
```

**Principles:** SuperMemo 1–3; pack first-principles card type.

---

### D4. Analogy (Rc)

**Bad**

```text
Q: What is Rc?
A: Reference counting.
```

**Good**

```text
Q: **Rc** analogy — family-room TV

A: **Rc<T>** is like a TV in a shared room: the first person turns it on; others can join and watch; the **last** person to leave turns it off.

- **Clone** of Rc = another person entering (refcount up).
- Drop last Rc = last person leaves (data freed).
- Turning it off early while others watch = invalidating remaining owners → **don’t** free while strong count > 0.

Maps to: **multiple owners**, free when **strong count hits zero**. **Single-threaded** only (like one living room, not multi-process).
```

**Principles:** SuperMemo 6 (imagery) + 14; pack intuition; say scope (**single-threaded**).

---

### D5. Elaborate this code (list with Box fails for two heads)

**Bad**

```text
Q: Can two lists share a tail?
A: Sometimes.
```

**Good**

```text
Q: Elaborate — why does this fail with **Box**?

enum List { Cons(i32, Box<List>), Nil }
let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
let b = Cons(3, Box::new(a));
let c = Cons(4, Box::new(a)); // error

A: **Cons** **owns** its tail. Creating **b** **moves** **a** into **b**. Using **a** again for **c** is a use-after-move.

- Lifetimes on references would force every element to live as long as the whole list — often wrong.
- **Rc<List>** on the tail: **clone** the Rc so **a** and **b** **share** ownership; refcount tracks last user.

Need: two list heads, one shared tail.  
Constraint: single ownership of Box.  
Mechanism: **Rc** for shared ownership (single-threaded).
```

**Principles:** mechanism card; SuperMemo 1; domain-true code story (Rust Book style).

---

### D6. Three-way contrast (Rc / Box / RefCell)

**Bad** — three separate vague definitions that interfere.

**Good**

```text
Q: **Rc** vs **Box** vs **RefCell** — owners and borrows?

A: One decision table:

- **Owners:** **Rc** = many; **Box** and **RefCell** = single owner of the wrapper.
- **Borrow checks:** **Box** — imm or mut, **compile time**. **Rc** — only **imm** borrows, compile time. **RefCell** — imm or mut, **runtime** (panic if rules break).
- **Mutate under shared ref:** only **RefCell** (interior mutability) among these three.

Combo: **Rc<RefCell<T>>** = many owners **and** runtime-checked mutation (single-threaded).
```

**Principles:** SuperMemo 11 (combat interference); pack contrast matrix.

---

### D7. Process / design order (DDD / microservices)

**Bad**

```text
Q: How do you design microservices?
A: Split the monolith into services.
```

**Good**

```text
Q: First step when designing a **microservice** (domain view)?

A: Resist starting from “what the service does.”

Start from **data / facts**: coupling, dependencies, **integrity constraints** that must hold from a business view.

Behavior and API shapes come **after** you know what must stay consistent and who owns which writes.
```

**Sister card (events-first):**

```text
Q: Main problem with starting DDD only from **nouns** (domain objects)?

A: You lock **structure** too early and miss **how change propagates**.

Events-first: begin with meaningful **things that happened** (past tense: OrderPlaced). From events derive commands, aggregates, and read models. Focus moves to **flow and communication**, not only entity fields.
```

**Principles:** process cards; SuperMemo 2–3 (picture and basics); pack “order of attack.”

---

### D8. Aggregate / unit of consistency

**Good**

```text
Q: What is an **aggregate** as a unit of consistency?

A: An aggregate is a cluster of entities treated as **one unit for data changes** — and therefore also a **unit of failure**: it fails, upgrades, and relocates **atomically**.

- One **aggregate root** is the only entry for modifications.
- Keeps invariants inside the boundary; other aggregates interact via the root’s API / messages, not by poking internals.
```

**Principles:** definition spine + why (atomicity of failure); moderate depth.

---

### D9. Concurrency — relevant example

**Bad**

```text
Q: What is a race condition?
A: Two people grabbing the same pan in a kitchen.
```

**Good**

```text
Q: Why can two **HTTP requests** on one server corrupt a shared in-memory counter?

A: Both handlers can **read** the same value, **add one**, and **write** back; the later write drops the earlier update.

- Need: exclusive (or atomic) update of shared state.
- Constraint: many concurrent tasks, one process memory.
- Fix shape: **mutex** / **atomic**, not “be careful in code review.”
```

**Principles:** SuperMemo 14 (personalize); pack domain-true examples.

---

### D10. Interior mutability / compile vs runtime

**Good**

```text
Q: Borrow rules: **compile time** (references / Box) vs **runtime** (**RefCell**)?

A: Compile-time checks: errors early, **no** runtime borrow tax — Rust’s default.

Runtime (**RefCell**): allows patterns the compiler cannot prove (e.g. mutate through **&self** when you know borrows don’t overlap). Cost: **panic** if you break the rules; small runtime bookkeeping.

Use RefCell when you are sure the dynamic pattern is sound and the compiler is too conservative — not as a shortcut around real data races (still **single-threaded** for classic RefCell).
```

**Principles:** first-principles tradeoff; interference-safe vs “what is interior mutability?”.

---

## E. Mini-deck sketch (how many of each)

After a smart-pointers chapter, a healthy mix might look like:

| Type | Example fronts |
|------|----------------|
| Why / first principles | Why Box for recursive types? |
| Intuition / analogy | Rc as family-room TV |
| Contrast | Box vs Cell; Rc vs Box vs RefCell |
| When to use | When Box? When Rc? |
| Code elaborate | Two list heads with Box vs Rc |
| Process | (if design chapter) start from data/events |

Roughly: few pure definitions, many **contrast / when / why / code**.

---

## F. Checklist against these sources

Before shipping a card, ask:

| Check | SuperMemo / Anki | Pack |
|-------|------------------|------|
| Understood? | Rules 1–2 | First principles pass |
| One idea? | Rule 4 | Atomic idea, moderate A OK |
| Sharp vs similar cards? | Rule 11 | Contrast fronts |
| Example real for learner? | Rule 14 | Domain-true table |
| Short front, scannable back? | Rule 12 | Lead + bullets + bold |
| Not a set dump? | Rules 9–10 | When-to-use / split |
| Why present for designs? | — | Intuition bar |

---

## G. Sources

| Source | URL / note |
|--------|------------|
| SuperMemo 20 rules | https://www.supermemo.com/en/blog/twenty-rules-of-formulating-knowledge |
| Atomic / precise Anki practice | Community guides: one precise cue; split when grading is ambiguous |
| High-yield / understand-first Anki | Med and systems blogs: foundations first; understand then memorize |
| Personal deck style | Quizlet-style Rust / DDD / systems cards: contrast, when, analogy, code elaborate, process |
| Pack | `guide.md`, `html-deck.md`, `explain-topic` study-card patterns |
