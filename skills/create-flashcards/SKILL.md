---
name: create-flashcards
description: >
  Create high-quality study flashcards as a self-contained HTML deck page: always-
  visible Q/A, every card has Copy (formatting preserved), important text in bold.
  Builds intuition and why — not bare fact lists. Use when the user asks for
  flashcards, Anki/Quizlet cards, a study deck, spaced-repetition cards, or to
  turn notes/a chapter/a talk into review cards. Load first-principles and
  intuition rules before drafting. Prefer moderate answers and domain-true
  examples. Not for full explain-topic lessons alone or multi-lens code review.
---

# Create flashcards

Turn material into a **visual HTML study deck** that rebuilds **context and intuition** on review. You are not writing a glossary of slogans.

**Learner bar:** prefer understanding **why** and a usable mental model over one-line facts. Cards may be **moderate length**. Language stays **simple and plain**.

**Default deliverable:** self-contained **HTML page** under `~/explanations/YYYY-MM-DD-flashcards-<slug>.html` — see `references/html-deck.md`.

**Full craft:** `references/guide.md` (required for non-trivial decks).  
**Examples + SuperMemo/Anki map:** `references/examples.md` (load when drafting or reviewing cards).  
**HTML / copy / bold:** `references/html-deck.md` (required when building the page).

## First principles (before any card)

Do not cardize what is not understood. For each candidate idea:

1. **Need** — what fails without this?  
2. **Constraints** — what you cannot assume?  
3. **Mechanism** — smallest shape that meets the need.  
4. **Drop if smaller need** — what goes away if the problem shrinks?  
5. **Wrong default** — what people try that breaks?

If (1)–(3) are thin → research or explain first (`explain-topic`), then cardize.

## Intuition (protect on every card that matters)

Prefer cards that force a **picture**:

- why it exists / what fails without it,  
- one **familiar domain** walkthrough (web requests for concurrency; orders for aggregates),  
- **contrast** and **when to use**,  
- **elaborate this code** when the model must survive contact with the compiler/runtime.

Sparse pure definitions. Heavy intuition, contrast, and judgment.

## Cover what people get wrong, not only what the source says

The source tells you what is **true**. It does not tell you what people **believe that is false**, what they **see when it breaks**, or **where they run into it**. A deck built only from the source can explain everything well and still leave the learner unable to spot the problem in real life.

So go through the ideas a second time and ask three questions:

1. **What does a sensible person believe here that is wrong?**
2. **What will they actually see on screen when this goes wrong?**
3. **Where will they run into it — in their code, their tools, their job?**

Each real answer becomes a card, and it goes on the **front**:

- **Wrong belief** → ask the wrong question so the learner has to reject it. *“How many bits is Unicode?”* → **“None. Unicode is not an encoding.”**
- **What they see** → put the broken output itself on the front. A line of garbled text → **“something read these bytes with the wrong table.”** This teaches them to recognize it, not just to explain it.
- **Where they hit it** → *“My app is UTF-8 everywhere, but the database admin page shows garbage.”* → **“The database is set to latin-1.”**

Explaining the idea correctly somewhere else in the deck does **not** do this job: being told the right answer and having to reject the wrong one are different things. These fronts often come from **outside** the source — the learner's own tools, common bug reports, what people argue about. That is where they should come from.

## Workflow

1. **Ingest source** — notes, chapter, talk, code, prior explanation, or user paste.  
2. **Model pass** — first principles, then outline the *ideas*. **Scale count to the material:** a light conceptual talk may need ~8–15; a **dense technical chapter warrants one atomic card per load-bearing item** — each key **term, unit, quantity, component, law, and named example** — which can total **25–40**. Enumerate with the coverage sweep in `guide.md`; skip only true trivia, never core vocabulary.  
3. **Draft cards** using the types below (craft in `guide.md`; **mirror patterns in `examples.md`**).  
4. **Edit** — plain language; one idea; moderate A; **relevant** examples; mark **important** phrases for bold.  
5. **Self-check** — guide § Self-check + examples § F + html-deck checklist.  
6. **Build HTML deck** — themed sections; every card Q+A visible; **Copy on every card**; bold important text; path under `~/explanations/`.  
7. **Handoff** — absolute path, card count, one-line summary (not a full dump of all cards in chat).

## Card types (emit a mix)

| Type | Front cue | Back must include |
|------|-----------|-------------------|
| **Why it exists / first-principles** | Why does X exist? What problem does it solve? What did people do before? | The real need and what breaks without X — **the backbone of the deck; use it a lot** |
| **Plain definition** | A precise term on its own (**Mutex**, **Pin**, **Bandwidth**) | Crisp definition + why it matters; **expand acronyms**, note a naming origin when it helps |
| **Contrast** | X vs Y | Decision table or sharp difference; a plain relational statement counts (**“ASCII is an English-language subset of Unicode”**) |
| **Misconception** | A question built on the **wrong belief** (“How many bits is Unicode?”) | Reject the premise in the first words, then give the right model |
| **Symptom → diagnosis** | The raw broken thing itself — garbled text, an error line, a wrong number | What it means and the single cause; trains recognition in the wild |
| **When to use / how-to** | When X? How do you do X? | Triggers + one “not when”; or the practical steps |
| **Intuition** | Picture / analogy / “how does this work?” | Portable model + where analogy breaks if needed |
| **Elaborate this code** | A code snippet → “why does this work / fail?” | Step through what the compiler/runtime does, and why |
| **Overview** | How does the whole thing work? | One high-level chain end to end, before the parts |
| **Scenario / apply it** | A concrete situation, then a choice — **including a real bug from outside the source** (“app is UTF-8, database admin shows garbage”) | Apply the concept; often options with pros/cons |
| **Worked number** | Compute it (use the formula) | Real values plugged in → the answer, then the intuition |
| **Enumeration** | Name N … | The short list, each item one line + a word of why |
| **Connection / bonus** | How does this relate to X you already know? | A link to another field, tool, or everyday thing (or an extra insight/analogy) — take teaching opportunities |

## Front / back rules (always)

**Front** — one clear question (contrast, when, why, what happens, elaborate). Not “discuss X.” Do **not** tack on redundant qualifiers like “(plain)”, “(plain walk-through)”, or restate the deck/section theme in the question.

**Self-contained (always)** — fronts *and* backs must stand alone. Never reference the source (“the book”, “this chapter”, “the author”, “we open with…”) or assume the reader has it open. Rewrite source-anchored prompts as concept questions — e.g. “What do Morse, Braille, and blinking lights share with a computer?”, not “Why does the book open with…?”. Test: could someone who never saw the source answer it? If it names the source, rewrite.

**Back — teach it like you'd say it out loud, not a bullet dump.** This shape (derived from the user's own decks):

1. **Answer first** — one or two crisp sentences that directly answer (the definition or key claim), key term in **bold**. Make it a sentence the learner could **say back in one breath**, and pick the wording that stops the most likely mistake rather than the most complete wording: *“Unicode is a big table mapping characters to numbers; the UTF encodings say how those numbers turn into bits”* beats a fuller definition that leaves the reader still thinking Unicode is an encoding. **Expand acronyms** (**mpsc** = multiple producer, single consumer) and note a naming origin when it helps (**Cow** = Clone-On-Write).  
2. **Then the deeper idea in flowing prose** — build the *why*: what problem it solves, what breaks without it, what people did before. This first-principles angle *is* the point.  
3. **Analogy when it lands** — name it, map it, note where it breaks (a channel is a river you drop a rubber duck into; a mutex is a panel with one microphone).  
4. **Make it concrete** — a short code walkthrough, worked numbers, or a real example when that makes the model load-bearing.

**Lists are for real lists** — pros/cons (✅/❌ is fine), options with trade-offs, ordered steps, or “name N” enumerations. In a **longer** card a genuine section header is fine (**The core idea**, **Why it exists**, **Takeaway**, **Pros/Cons**). What's banned is chopping one flowing thought into `Label:` stage-narration (`Push:`, `Flow:`, `Work:`, `Need:`, `Constraint:`). Include the canonical **slogan** when one exists (“Do not communicate by sharing memory; instead, share memory by communicating”).

**No reflexive trailing bullets.** Do not staple a 2–4 item bullet summary onto the end of every card. Most cards should be **pure prose** — many good cards have **zero** bullets. Before adding a list, apply the **bullet test**: would these items still read fine as sentences, or do they just restate the prose above? If either is true, **delete the bullets** or fold them into the prose. Keep bullets only for a genuine parallel set (options, pros/cons, ordered steps, “name N”) where the parallel structure itself carries meaning — not as a habitual closer.

**Bold (page + copy payload):** type names, critical outcomes, constraints, contrast poles — not whole paragraphs. See `html-deck.md`.

**Length follows the idea, not a house style.** One focused screen is the **ceiling, not the target**. When the idea genuinely is one line, write one line — *“ASCII: 7 bits = 1 character.”* / *“UTF-32: every character in 32 bits; the encoding is just the code point. Downside: bloated.”* A deck where every card is four paragraphs has no anchors, flattens what matters, and is punishing to review. Mix short, moderate and long; let the hard ideas be the long ones.

**Atomic:** one *idea* per card.

## Examples must be relevant

| Topic | Prefer | Avoid |
|-------|--------|-------|
| Concurrency | Concurrent HTTP requests, shared pool | Random kitchen scene as the only model |
| Shared ownership | Graph node, shared list tails | Unrelated sports |
| Consistency / DDD | Order + line items | Vague “teamwork” |

## HTML deck (default visualization)

| Requirement | Detail |
|-------------|--------|
| File | `~/explanations/YYYY-MM-DD-flashcards-<slug>.html` |
| Cards | Always-visible Q and A |
| **Copy** | **Every** card has a Copy button. It must paste as **formatted rich text (real bold/code), never markdown.** Write two clipboard flavors: `text/html` (real `<strong>`/`<code>`, line breaks, bullets) + a clean `text/plain` fallback with `**`/backticks **stripped**. Keep newlines, blanks, and bullets in both. |
| **Bold** | `<strong>` on important terms in the page |
| Self-contained | Inline CSS/JS only |
| No flip UI | Unless the user explicitly asks |

Implementation pattern and CSS: **`references/html-deck.md`** (load when writing the file).

**Chat-only markdown** only if the user says so; still keep the same craft rules. Prefer HTML even then if they want copy buttons.

## Do (quality bar)

- Understand → then cardize.  
- Prefer **why / intuition / contrast / when** over definition spam.  
- **Ask the wrong question at least a few times per deck** — misconception, symptom and real-bug cards, drawn from outside the source when needed.  
- **Vary the length.** Some cards are one line. If every card looks the same size, the deck has no anchors.  
- **Lead with why-it-exists and analogies** — that is how the user's own decks build intuition, not memorization. Expand acronyms; include the canonical slogan when one exists.  
- **Take every teaching opportunity.** When there's a chance to be educational — an extra insight, a connection to another field or to something the learner already knows (databases, code, everyday life), or a clarifying analogy — take it. Add it as a short woven paragraph, or as its own **bonus / connection** card when it's a distinct idea. Keep it plain and relevant; do not pad.  
- Plain language; moderate meaningful backs.  
- **Plain, ordinary words only** — no coined pattern nicknames or invented metaphors.  
- **Domain-true, concrete examples**: prefer the source’s own canonical example (names, shape, story) over a weaker invented one; add a short **code walkthrough** when the model must survive the compiler/runtime.  
- For a **failure/bug** card, add a companion **“how do you prevent it?”** card so problem and fix sit together.  
- HTML page with **copy on every card** and **bold** on important text.  
- Split independent ideas; keep why on the claim.

## Do not

- Deck of one-line jargon definitions as the main product.  
- **Coined pattern nicknames / invented metaphors** in cards (e.g. “safe disciplines”, “the one brick”, “blessed path”, “sanctioned”). Use plain words.  
- **Redundant qualifiers in the front** (“(plain)”, “(walk-through)”) or **visible tag chips** that repeat words already in the question/section — keep tags for the search filter only (`data-search`), do not render them.  
- **Leak first-principles scaffolding into card text.** The `Need:` / `Constraint:` / `Mechanism:` / `Drop it:` / `Wrong default:` analysis stays in your **notes** — never render those labels in a back. (Before shipping, scan every back for those words and delete them.)  
- **Default to bullets / chop an explanation into fragments.** A back should read as **flowing prose** (a short story). Reach for bullets only for a real list or a sharp contrast — not as the standard shape. In particular, do **not** end every card with a 2–4 bullet “summary” — that reflex is the most common failure; if the bullets could be sentences or just echo the prose, cut them.  
- **Reference the source in a card** (“the book/chapter/author”, “we open with…”) or write a front that only makes sense with the source in hand — every card must stand alone.  
- Skip the HTML page when building a real deck (unless chat-only).  
- Cards without a working Copy control.  
- **Copy that pastes markdown** (literal `**`/backticks) or loses spacing — see `html-deck.md`.  
- Flip / hide-answer UI by default.  
- Irrelevant toy stories for serious systems topics.  
- Cardize material you have not understood.  
- Full explain-topic lesson page unless asked (link that skill for long teaching).

## Related

| Job | Skill |
|-----|--------|
| Long HTML mental model + optional study block | `explain-topic` |
| Diff intuition | `explain-diff` |
| Standalone visual deck + copy | `create-flashcards` |
