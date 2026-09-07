# Writing flashcards for understanding

How to turn material into **study cards** that build **context, intuition, and why** — not a pile of disconnected slogans.

This pack’s goal (from how you already learn): when studying, you are not after bare facts. You want a **mental model** — why the idea exists, how it fits, what fails without it, a picture you can reuse. Cards can be **moderate length** when the meaning needs it. Short is not the same as empty.

Classic spaced-repetition advice still matters for *form*. Your preference overrides it when the two conflict: **understand first; moderate answers beat one-line trivia.**

---

## Two layers of craft

| Layer | Job |
|-------|-----|
| **Learning order** | Understand the whole → then encode pieces. Never memorize what you do not understand. |
| **Card form** | One clear idea per card; Q that forces recall; A that answers with why + useful context; plain language; **relevant** examples. |

Internet sources (SuperMemo 20 rules, Anki communities) stress **minimum information** (one fact, tiny cards). That optimizes *grading* and *scheduling*. It is a good default for pure facts (vocab, constants).

For **systems and design** (Rust ownership, DDD, latency), your decks show a better default: cards that still **fit on one screen**, but carry **enough** of the model that the answer rebuilds intuition — analogy, comparison, “when to use,” “why not the other type,” a short code story.

---

## First principles (how to think before writing any card)

Do this **before** drafting Q/A. Same spirit as explain-topic / pack first principles — applied to each *idea* you might turn into a card.

For each candidate idea, write (in notes, not necessarily on the card):

1. **Need** — what job fails without this idea?  
2. **Constraints** — what you cannot assume (single owner, single thread, must fail closed, …)?  
3. **Mechanism** — smallest rule or type shape that meets the need under those constraints.  
4. **What you would drop** if the need were smaller.  
5. **Wrong default** — what people try first that breaks, and why.

If you cannot answer (1)–(3), **do not make a card yet**. Read, diagram, or explain the topic first (optionally via `explain-topic`). Flashcards encode understanding; they do not create it.

Also:

- Prefer **why** and **mechanism** over slogans (“best practice,” “use X”).  
- Restate the real need in plain words; strip inherited jargon until it is necessary.  
- If a card could be swapped for “because that’s how the book says it” with no loss of meaning, rewrite until the need is visible.

---

## Intuition (what every good deck must protect)

**Intuition** = the portable picture of how something works — small enough to hold in mind, strong enough to predict what happens next.

A card protects intuition when the answer:

- paints a **concrete situation** (not only a definition),  
- shows **why** that design appears under constraints,  
- uses a **relevant** example the learner already knows in that domain,  
- optionally **contrasts** with the near alternative (Box vs Rc, noun-first DDD vs events-first).

### Intuition card patterns (prefer these)

| Pattern | Question shape | Answer does |
|---------|----------------|-------------|
| **Why it exists** | “Why does X exist?” / “What fails without X?” | Need → failure mode → shape of the solution |
| **Toy value through the design** | “How is *this* value represented / owned?” | Walk one familiar value (e.g. `"hello"`, one order, one open connection) through the layout |
| **One operation** | “What happens when you move / clone / await / place an order?” | Steps; who owns what after |
| **Contrast** | “X vs Y?” / “When X not Y?” | Constraints → different answers for the same job |
| **When to use** | “When do you reach for X?” | 2–4 real triggers; one “not when” |
| **Analogy that lands** | “Rc analogy?” | One tight picture; say where it breaks if needed |
| **Elaborate this code** | Short snippet on the front | What the compiler/runtime does; why it fails or works |
| **First step / order of attack** | “First step when designing a microservice?” | Principle of order (data/facts before behavior), not a tool list |

### Intuition anti-patterns

| Avoid | Prefer |
|-------|--------|
| Glossary-only: “What is SSO?” → one jargon sentence | “Where do the bytes of `"ok"` live with SSO vs heap?” |
| Random toy domain for a systems idea | Domain-true story (web server + concurrent requests for concurrency; family TV for Rc — *if* it maps) |
| Mega-card that dumps a whole chapter | Several cards: plain definition → when to use → contrast → one code story |
| Metaphor that is cuter than accurate | Accurate short mechanism; metaphor only if it carries the constraint |

---

## What your decks already do well (derived principles)

From sets like **Rust smart pointers**, **DDD / events-first**, **latency**, **sharding**:

1. **Front can be a term, a contrast, a “when,” or a code puzzle** — not only “Define X.”  
   - *Box vs Cell*, *Rc vs Box vs RefCell*, *When to use Box?*  
2. **Backs teach** — start with the main point, then the cases in prose, then a “why compile time vs runtime” style payoff.  
3. **Comparisons are first-class** — three-way tables of ownership/borrow checking are more useful than three isolated definitions.  
4. **“When to use” cards** turn knowledge into judgment.  
5. **Analogies are short and structural** (TV in the family room for Rc) — they encode *shared use until last owner leaves*, not fluff.  
6. **Code elaborations** make the model load-bearing (why `a` cannot feed two `Cons` with `Box`).  
7. **Order-of-design cards** capture process (events before nouns; data coupling before service behavior).  
8. **Moderate length** — longer than Anki trivia, shorter than a blog post; scannable bullets.

**House rule for this skill:** match that bar. If a card is only a slogan, expand or delete.

---

## Classic formulation rules (keep, with your overrides)

From SuperMemo’s *20 rules of formulating knowledge* and Anki practice — **adapted**:

| # | Rule | Your override |
|---|------|----------------|
| 1 | Do not learn what you do not understand | Same — non-negotiable |
| 2 | Learn before you memorize | Same — build picture first |
| 3 | Build on basics | Same — foundations before edge cases |
| 4 | Minimum information | **One idea per card**, but the answer may be **moderate** if that idea needs why + context |
| 5 | Cloze / delete carefully | Prefer full Q/A for mechanisms; cloze OK for lists of *named* steps |
| 6 | Avoid mega-sets without structure | Prefer “when to use” / ordered steps / one-item-per-card rather than 12-bullet walls |
| 7 | Combat interference | Similar cards must differ by a sharp contrast (Box vs Rc, not two soft definitions) |
| 8 | Personalize with **your** domain examples | Always — web server, orders, nodes, not “Alice’s banana” unless teaching toddlers |

**Atomic** here means **one questionable idea**, not **one short sentence**.

Split if:

- the answer has two independent guarantees,  
- grading “half right” is common,  
- the front asks two unrelated things.

Do **not** split if:

- the second sentence is *why* for the first,  
- bullets are facets of one contrast (Rc vs Box vs RefCell),  
- removing context makes the card useless later.

---

## Front (question) quality

Good fronts **force active recall** of a useful idea.

| Strong front | Weak front |
|--------------|------------|
| Box vs Cell | Smart pointers (too broad) |
| When to use Rc? | Tell me about Rc |
| Why events-first instead of noun-first DDD? | What is DDD? |
| Elaborate: recursive List with Box (snippet) | What is recursion? |
| How can multiple owners mutate shared data in single-threaded Rust? | RefCell |

Rules:

- One prompt; one primary answer.  
- Prefer **contrast**, **when**, **why**, **what happens when**, **elaborate code**.  
- Put **stable cues** on the front (type names, pattern names) so later review still makes sense.  
- Avoid “list everything you know about X.”

---

## Back (answer) quality

**Write the back as a short teaching *story* in prose — not a bullet list.** This is the default shape:

1. **First sentence** — the main point, key claim in bold.  
2. **Flowing explanation** (2–5 short sentences) that builds the picture and the *why*, with a concrete example woven into the prose.  
3. **Optional close** — a consequence or “not when.”

**Prose is the default; bullets are welcome where they genuinely help** — a real **enumeration** (silver / copper / gold; the three particles of an atom), a **sharp contrast** (fixed vs variable), or a few clear **steps** — just not as the automatic shape for every card. Do **not** chop one explanation into bullet fragments, and never write `Label: text` narration (`Push:`, `Flow:`, `Work:`) — those read as scaffolding, not teaching. If bullets are just the steps of a single thought, turn them into sentences.

**No reflexive trailing bullets.** The most common failure is stapling a 2–4 item bullet "summary" onto the end of every card. Do not do this. Most cards should be **pure prose with zero bullets**. Before you add a list, apply the **bullet test**: would these items still read fine as sentences, or do they just restate the prose above? If either is true, delete the bullets or fold them into the prose. Keep bullets only when the parallel structure itself carries meaning (options, pros/cons, ordered steps, "name N") — never as a habitual closer.

Model the tone on how a person explains out loud: “Current is like the amount of water flowing through a pipe, voltage is the pressure pushing it, and resistance is how skinny the pipe is — so more pressure means more flow, and a skinnier pipe means less.” That is one flowing card, not five bullets.

**Language:** plain words; common word first; jargon defined on first use. No corporate padding.

**Length:** roughly **one screen** of focused prose (a short paragraph or two — think 3–8 sentences). Too short if the “why” is missing. Too long if it rewrites the whole chapter — split.

**Code on the back:** only when the point is how the compiler or runtime treats that shape. Keep snippets minimal; explain the *failure or win* in words.

---

## Relevant examples (non-negotiable)

Examples must help **this** learner with **this** topic.

| Domain | Prefer | Avoid |
|--------|--------|-------|
| Concurrency | Web server handling many requests; shared connection pool | Random “two people in a kitchen” unless you only need a soft intro |
| Rc / shared ownership | Graph edges → node; multiple list heads sharing a tail | Unrelated sports analogies |
| RefCell / interior mutability | Mock object counting calls; UI tree needing mutate under `&self` | Abstract “sometimes you need mut” |
| Aggregates / consistency | Order + lines that must commit together | Vague “team collaboration” |
| Latency | Client round-trip, p99, fan-out to dependencies | Pure math with no system |

**Canonical teaching examples first** when a standard text already has the classic demo (Rust Book ownership/`String`, etc.) — then domain examples the learner lives in.

Fail bar: if the example could be swapped for any other topic without changing the sentence, it is decoration — rewrite.

---

## Card types to emit (mix in a deck)

Aim for a **balanced deck** after a chapter/talk:

| Type | Share (rough) | Purpose |
|------|----------------|---------|
| **First-principles / why** | Some early | Need and constraints |
| **Intuition / picture** | Heavy | Portable model |
| **Contrast / when** | Heavy | Decision skill |
| **Mechanism / elaborate code** | As needed | Make model load-bearing |
| **Plain definition** | Sparse | Only when a term is load-bearing and easy to misuse |
| **Process / order** | When design method matters | “Start with events / data, not service behavior” |

Do **not** emit a deck of pure definitions. But for a **technical** chapter, do **not** under-cardize either: a bare definition is bad, a *missing* one is worse. Give each core term its own card **with** a why/example.

---

## Coverage & granularity (technical material)

Few dense synthesis cards feel elegant but leave the learner unable to recall the pieces. For a foundations/technical chapter, prefer **broad atomic coverage**: sweep the source and make a card for **each load-bearing item**.

**Coverage sweep — one card (often two) per:**

| Item in the source | Card(s) to emit |
|--------------------|-----------------|
| Each **key term** (conductor, insulator, resistor) | definition + *why / how it works* |
| Each **unit / quantity** (volt, amp, watt, ohm) | what it measures + a scale example |
| Each **component / part** (battery, filament, switch) | its **purpose** + its **mechanism** |
| Each **law / relationship** (Ohm's law) | the formula + a **worked-number** card + the intuition |
| Each **analogy the source uses** (water in pipes) | one card, incl. the proportional relationship and where it breaks |
| Each **“why does X work / why not Y”** (series vs reversed cells) | a why/contrast card |
| The **whole system** | one **overview** “how does it all work” card, before the parts |
| The **reason it exists** | one **purpose** “why do you even need X” card |

**Second pass — what the source does not contain.** The table above only finds cards the source suggests, so it can only ever produce a deck that explains the source. Go through the ideas again from the learner's side:

| Ask | Card to emit |
|-----|--------------|
| What does a sensible person **believe that is wrong**? | A question built on the false belief, answered by rejecting it first (“How many bits is Unicode?” → “None. Unicode is not an encoding.”) |
| What will they **see on screen** when it breaks? | The broken output on the front, the cause on the back (a line of mojibake → “read with the wrong table”) |
| Where will they **run into it** in real work? | A concrete situation from their stack, even if the source never mentions it (“app is UTF-8, database admin shows garbage” → “the database is latin-1”) |
| What is the **one sentence** they should be able to repeat? | A short card whose whole answer is that sentence — no build-up, no paragraphs |

These are usually the cards a learner remembers, because they match how the knowledge actually gets used: you meet a symptom, not a chapter. Expect a handful per deck, and do not skip them because the source never raised them.

**Granularity rule:** one card = one *questionable* idea. If a term, a unit, and a law appear together, that is **three** cards, not one “voltage/current/resistance” mega-card (mega-cards cause interference and half-right grading). Count follows concepts: a dense chapter is routinely **25–40 cards**.

**Worked-number cards:** for every formula, add a card that plugs in **real values** and shows the result, plus a contrast of extremes (e.g. 1.5 V through air → R huge → I ≈ 0; through a copper short → R tiny → I huge). Numbers make the model load-bearing.

---

## Growing an existing deck (incremental adds)

Decks are built up over a reading, card by card. When adding:

- Keep **one source-of-truth data array** (e.g. a `CARDS` list) that drives both the visible card and the copy payload — never hand-duplicate Q/A.
- **Validate after each edit** (open the file, or parse the embedded data) so a stray quote never silently breaks the page.
- **Guard against near-duplicates:** before adding, check the new card against existing ones. If it blurs into another, either **sharpen it into a contrast** or **fold** the point into the existing card. Twins without a sharp difference cause interference (SuperMemo rule 11).
- Place the new card under the **right theme**; add a new theme section if a cluster forms (e.g. batteries, conductors).

---

## Workflow (agent)

1. **Source** — notes, book chapter, talk, code, or “from this explanation page.”  
2. **Understand** — first-principles pass; if thin, research or refuse to cardize slogans.  
3. **Outline the model** — 5–15 *ideas*, not 50 facts.  
4. **Draft cards** by type (why / intuition / contrast / when / code / process).  
5. **Edit** for plain language, relevant examples, one idea, moderate length; mark phrases that must be **bold**.  
6. **Self-check** (below).  
7. **Deliver HTML deck** — default visualization (see `html-deck.md`): themed page, always-visible Q/A, **Copy on every card** that pastes as **formatted rich text, not markdown** (real bold/code via `text/html`, clean `text/plain` fallback; newlines + bullets kept), important text in `<strong>`.  
8. Path: `~/explanations/YYYY-MM-DD-flashcards-<slug>.html`. Chat: path + count + one-line summary only.

### Output shape (default = HTML)

Self-contained HTML page. Each card:

- Q and A always visible  
- **Copy** button → **formatted rich text, not markdown** (see `html-deck.md` § "Copy = formatted text"). The card *source* below drives both flavors: `**bold**`/`` `code` `` become real `<strong>`/`<code>` in the `text/html` flavor and are **stripped** in the clean `text/plain` fallback.

```text
Q: Box vs Cell — when each?

A:
**Box<T>** is single ownership of heap data (or a fixed-size handle to unsized/recursive data).

**Cell<T>** is interior mutability for **Copy** values: mutate through `&T` when you need a simple replace/get, not shared ownership.

- Reach for **Box** when the issue is where data lives / recursive size.
- Reach for **Cell** when the issue is mutating behind a shared reference for Copy payloads.
```

- On the page, the same important words use `<strong>…</strong>`. Paste-test in a rich editor: the result must show **real bold**, never literal `**`.

Markdown-only or CSV/YAML only if the user explicitly wants that instead of (or in addition to) the HTML page.

---

## Self-check (before shipping a deck)

For **each** card:

1. Could you answer from **understanding**, not from memorizing the sentence?  
2. Does the answer include **why** or a **constraint**, not only a label?  
3. Is the example **domain-true** for this topic?  
4. One idea? If “and” joins two tests, split.  
5. Front is a **real question** (contrast / when / why / what happens), not “discuss X”?  
6. Plain language — **ordinary words, no coined nicknames/metaphors** (e.g. not “safe disciplines”, “the one brick”)?  
6a. Front has **no redundant qualifier** (“(plain)”, “(walk-through)”) and does not restate the deck/section theme?  
6b. **Self-contained** — no reference to the source (“the book/chapter/author”); answerable without it in hand?  
6c. **No scaffolding words** in the back — search for `Need:` / `Constraint:` / `Mechanism:` / `Drop it:` / `Wrong default:` and remove?  
6d. **Reads as flowing prose (a short story)**, not a chopped bullet list or `Label: text` narration? Bullets only for a real list/contrast — **not a reflexive 2–4 item summary stapled onto the end**? Apply the bullet test: if the items would read fine as sentences or just restate the prose above, delete or fold them.  
7. Moderate length — not a tweet, not a chapter?  
8. Important terms marked for **bold** (page + copy payload)?  
9. **Copy** pastes as **formatted rich text, not markdown** (real bold in a rich editor; clean plain text with no `**`/backticks elsewhere), keeping newlines and bullets?

For the **deck**:

1. Early cards build the **picture**; later cards add edges.  
2. At least some **why** and **contrast** cards, not only terms.  
3. No two cards that blur into each other without a sharp difference.  
4. **Coverage:** for a technical chapter, did the sweep hit every key term, unit, quantity, component, law (with a worked-number card), and analogy — plus one overview and one purpose card?  
4a. **Second pass done?** At least a few cards where the front is a **wrong belief**, a **symptom the learner will see**, or a **real situation** — including ones the source never mentions. A deck with none of these explains well and prepares nobody.  
4b. **Length varies?** If every card is the same size, the one-line ideas got padded. Check that the simplest facts got the shortest answers.  
5. HTML file opens locally; no external assets; every card has Copy.

---

## Bad vs good (compressed)

**Bad**

```text
Q: What is RefCell?
A: A smart pointer for interior mutability.
```

**Good**

```text
Q: RefCell vs Box — where are borrowing rules enforced?

A: Box (and ordinary references): compile time. Break the rules → compiler error; no runtime borrow flag cost.

RefCell: runtime. Break the rules → panic. Use when you are sure the dynamic borrows are sound but the compiler cannot see it (e.g. mutate inside a method that only has &self, in single-threaded code).

Tradeoff: later errors and a small runtime cost, in exchange for patterns compile-time checking rejects.
```

**Bad example for concurrency**

```text
Q: What is a race?
A: Two people cooking and grabbing the same pan.
```

**Good example for concurrency** (if the learner builds services)

```text
Q: Why can two requests on one web server corrupt a shared in-memory counter without synchronization?

A: Both request handlers may read the same value, add one, and write back; the second write drops the first update.

- Need: exclusive update of shared state (or an atomic).
- Constraint: many tasks, one process memory.
- Fix shape: mutex/atomic around the counter — not “be careful.”
```

---

## Sources (formulation)

| Source | Use here |
|--------|----------|
| SuperMemo — [20 rules of formulating knowledge](https://www.supermemo.com/en/blog/twenty-rules-of-formulating-knowledge) | Full map + Dead Sea lesson: `examples.md` § A |
| Anki / atomic-card practice | `examples.md` § B |
| Personal-style decks (Rust, DDD, systems) | Worked good/bad cards: `examples.md` § C–E |
| Pack `explain-topic` study-card section | Intuition-first Q shapes; scannable answers |

**Load `examples.md` when drafting** — copy structures (contrast, when, analogy, elaborate code, process), not only slogans.

---

## Related pack skills

| Skill | Relation |
|-------|----------|
| `explain-topic` | Build the model first; optional study cards on the HTML page |
| `explain-diff` | Intuition before mechanism for a change (not a deck) |
| `create-flashcards` (this) | Standalone deck production for ongoing review |
