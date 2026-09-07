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
| 17 | **Useful redundancy** | Same fact from two angles (plain definition + when-to-use + contrast) is OK. |
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

**Conflict with pure minimum-info culture:** pure SuperMemo wants ultra-short answers. This pack’s learner wants **intuition and why**. Resolution: **one idea**, answer **moderate and scannable** (prose by default, bold on key terms; bullets only when the content is a real list or contrast), not a slogan and not a chapter.

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

Use these as **templates** when drafting. Bold = important on the HTML page (`<strong>`); keep `**…**` markers in the card *source* only — Copy turns them into **real bold** rich text and strips them from the plain-text fallback (see `html-deck.md` § "Copy = formatted text"). Pasted output must never show literal `**`.

> **Read the bullets in D1–D11 as earned, not as a template.** Several of these examples end with a short bullet list — but only because the content is a genuine list or contrast (two named options, pros/cons, ordered steps). **Do not staple a 2–4 item bullet "summary" onto every card.** That reflexive trailing summary is the single most common failure. Most cards should be **pure prose with zero bullets**. Before copying the *shape* of any example here, apply the **bullet test**: would the items still read fine as sentences, or do they just restate the prose above? If either is true, they don't belong. For the prose-first target style, look at **D12–D17** — they mostly carry the whole idea in flowing prose with no trailing list at all.

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

**Principles:** plain definition + why (atomicity of failure); moderate depth.

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

### D11. Technical vocabulary — granular + self-contained (electricity)

For a foundations chapter, cover **every** load-bearing item, each card standing alone (no “the book says…”), each definition carrying a why/example. This is the style to match.

**Bad** — one mega-card that interferes and grades half-right:

```text
Q: Explain voltage, current, and resistance from the chapter.
A: Voltage is pressure, current is flow, resistance fights it, and Ohm's law ties them together…
```

**Good** — split into atomic, self-contained cards, each answer a short story in **prose** (no bullet chopping):

```text
Q: What is an **amp** (ampere)?
A: An amp is the **unit of current** — it tells you **how much charge flows past a point each second**. More amps means more electrons streaming past, which means more **power** delivered: a small LED sips a few **milliamps**, while a toaster pulls several **amps**.
```

```text
Q: What is a **watt**, and why does it matter?
A: A watt is the unit of **power** — the **rate** at which energy is used, equal to **voltage × current**. It answers "how fast is energy being converted?", not how much in total, so a **100 W** bulb runs brighter and hotter than a **40 W** one. It's what sizes everything: how bright a bulb is, how much a device draws, and, over time, your electricity bill in **kilowatt-hours**.
```

**Good — worked-number card for a law (prose, numbers plugged in):**

```text
Q: By **Ohm's law**, what current flows from 1.5 V through **air** vs a **copper short**?
A: The push is the same 1.5 V both times, but the resistance is wildly different, and **current = voltage ÷ resistance (I = E / R)** decides the rest. Through **air** the resistance is enormous, so 1.5 divided by a huge number is **about zero amps** — the voltage is real but nothing flows. Through a **copper short** the resistance is tiny, so 1.5 divided by a very small number is **huge** — a flood of electrons, and the wire heats up fast. Voltage alone tells you nothing; **resistance decides** how much of it becomes current.
```

**Good — overview and purpose cards (emit one of each per system):**

```text
Q: **How does electricity work**, in one high-level picture?
A: In a metal, each atom holds its outer electrons **loosely**, so they can drift from atom to atom. A battery makes one terminal **electron-rich (−)** and the other **electron-poor (+)**, and that imbalance — the **voltage** — pushes those free electrons around any **closed loop** you connect. Their steady drift is the **current**, and where it meets **resistance** like a bulb's filament, the push turns into **heat and light**. Break the loop and it all stops: with no complete path, the voltage just sits there.
```

```text
Q: Why do you even need a **battery** in a circuit?
A: The battery is the **source of the push**. A wire is already full of free electrons, but with nothing driving them they just jiggle in place and go nowhere. The battery holds one end **electron-rich** and the other **electron-poor**, so there is a steady **voltage** pulling electrons around the loop — and it keeps topping up that imbalance, so the flow stays steady instead of fizzling out like a one-off static spark.
```

**Principles:** coverage sweep (term / unit / law / overview / purpose); self-contained fronts; prose answers that teach; worked numbers; no `Need:`/`Constraint:` labels or bullet-chopping.

---

## D12–D17. Exemplars in the user's own voice (match this)

These are lightly cleaned from the user's real decks (Rust, web servers, latency, sharding, Bayesian). They show the target voice: **answer first → deeper why → analogy → concrete → real lists only where needed.** Fronts are self-contained; acronyms expanded; canonical slogans kept.

### D12. Definition + analogy (answer first, then a picture)

```text
Q: **Channel** (concurrency)

A: A **channel** carries data from one thread to another. One half is a **transmitter**, the other a **receiver**; you send on one end and read on the other, and the channel is **closed** once either half is dropped.

Picture a directional stream of water: drop a rubber duck in upstream (send) and it travels to whoever is waiting downstream (receive).

Rust's std channel is **mpsc** = multiple producer, single consumer: many senders, one receiver.
```

### D13. Analogy card (name it, map it)

```text
Q: **Mutex** — analogy

A: A **mutex** (mutual exclusion) lets only **one thread at a time** touch the data. A thread must **acquire the lock** before access, and the mutex "guards" the data via that lock.

Think of a panel discussion with a **single microphone**: before you speak you ask for the mic, you hold it while you talk, then hand it on. If someone forgets to pass it back, **no one else can speak** — the classic deadlock.
```

### D14. Why it exists / first-principles (the backbone type)

```text
Q: Why does the **binomial distribution** exist — what problem does it solve?

A: It answers "how likely are **k successes in n independent tries**?" without you listing every sequence by hand.

Without it you'd enumerate all outcomes: for 3 heads in 10 flips there are 120 orderings (HHHTTTTTTT, HHTHTTTTTT, …). The formula collapses that counting into one expression — the **binomial coefficient** counts the orderings, and **p^k (1−p)^(n−k)** gives each one's probability.

Use it only when the four assumptions hold: **fixed n, independent trials, constant p, two outcomes.**
```

### D15. Elaborate this code (compiler/runtime walkthrough)

```text
Q: Elaborate — why doesn't this compile?

enum List { Cons(i32, Box<List>), Nil }
let a = Cons(5, Box::new(Nil));
let b = Cons(3, Box::new(a));
let c = Cons(4, Box::new(a)); // error

A: **Cons owns its tail.** Building `b` **moves** `a` into `b`, so using `a` again for `c` is a **use-after-move** — Rust allows only one owner.

References would force lifetime annotations (every element living as long as the whole list), which is usually wrong. The fix is **Rc<List>**: `clone` the Rc so `a` and `b` **share** ownership, and the refcount tracks the last user.
```

### D16. Scenario + options with pros/cons (✅/❌ is fine here)

```text
Q: A client must route a request to the right shard. How does it learn the **shard → node** mapping?

A: Four common options, trading simplicity for freshness:

- **Hardcoded list** — ✅ trivial, no infra; ❌ adding a node means a redeploy everywhere.
- **Static config file** (reloaded on signal) — ✅ scale without code change; ❌ must push it to every server in sync.
- **Central registry** (ZooKeeper/etcd/Consul) — ✅ single source of truth, real-time, handles failure detection; ❌ a SPOF and a hard dependency.
- **Gossip** (SWIM = Scalable Weakly-consistent Infection-style Membership) — ✅ no coordinator, resilient; ❌ eventually consistent, brief disagreement.
```

### D17. How-to (practical, mechanism-backed)

```text
Q: How do you make sure a **Mutex** is always released in Rust?

A: You don't call unlock — you let **scope** do it. `lock()` returns a **MutexGuard** smart pointer that **Derefs** to the inner data and, on **Drop** (end of scope), releases the lock automatically.

So the lock can't be forgotten. Watch the scope, though: `while let Ok(job) = rx.lock().unwrap().recv()` holds the guard for the whole loop body, serializing everyone; `let job = rx.lock().unwrap().recv().unwrap();` drops the temporary guard immediately, freeing the mutex before the work runs.
```

### D18. Misconception (ask the wrong question)

The strongest card type for a widely misunderstood idea. The front repeats the false belief as if it were reasonable; the back rejects it in the first word.

```text
Q: How many bits is **Unicode**?

A: **None — Unicode is not an encoding.** Unicode is a big table mapping characters to numbers. The **UTF** encodings (UTF-8, UTF-16, UTF-32) are what say how those numbers turn into bits.

So "how many bits" is only answerable once you name the encoding: in UTF-32 always 32, in UTF-8 anywhere from 8 to 32.
```

**Why it beats a correct explanation elsewhere:** a card that says "Unicode assigns numbers, encodings assign bytes" teaches the fact. This one makes the learner *reject* the belief they arrived with. Being told the right answer and having to reject the wrong one are different acts of memory.

---

### D19. Symptom → diagnosis (put the broken thing on the front)

```text
Q: You open a document and it reads:

ÉGÉìÉRÅ[ÉfÉBÉìÉOÇÕìÔÇµÇ≠Ç»¢

What is wrong?

A: **The program reading it assumed the wrong encoding.** That is the one and only cause — the bytes are almost certainly fine.

Nothing is corrupted and nothing needs repairing at the source. The text was written with one table and is being read with another, so each byte gets looked up in the wrong row. Point the reader at the right encoding and the text comes back intact.
```

**Principle:** the learner meets a symptom, not a chapter. Train the recognition, not only the theory.

---

### D20. Real bug from outside the source

Nothing in the book mentions databases. That is not a reason to skip it.

```text
Q: My site is UTF-8 end to end — the app handles UTF-8 and stores UTF-8 — and it works fine, but the database admin page shows garbled text. What is going on?

A: **The database is set to latin-1 while the app speaks UTF-8.**

The app writes UTF-8 bytes, the database stores them without complaint, and reading them back through the app works because the same wrong assumption cancels out. The admin interface is the one place that honestly applies the database's declared encoding — so it is the only thing telling you the truth.
```

**Principle:** cards should come from where the learner will actually stand, not only from where the author stood.

---

### D21. One-line anchors (do not pad these)

Some ideas are one line. Making them four paragraphs hides which ideas are hard.

```text
Q: How many bits is **ASCII**?
A: **7 bits = 1 character.**
```

```text
Q: **UTF-32**
A: The simplest encoding: every character in **32 bits**, so the encoding *is* the code point. Downside: bloated — four bytes for a letter that needs one.
```

```text
Q: **Unicode** vs **ASCII**
A: **ASCII is an English-language subset of Unicode** — every ASCII character exists in Unicode, at the same number.
```

**Principle:** length follows the idea. A deck of uniformly moderate cards has no anchors and flattens the difference between a hard idea and a small one.

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
| Short front, scannable back? | Rule 12 | Prose + bold; bullets only if a real list |
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
