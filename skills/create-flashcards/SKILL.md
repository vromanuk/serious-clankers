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

## Workflow

1. **Ingest source** — notes, chapter, talk, code, prior explanation, or user paste.  
2. **Model pass** — first principles + outline 5–15 *ideas* (not 50 trivia bits).  
3. **Draft cards** using the types below (craft in `guide.md`; **mirror patterns in `examples.md`**).  
4. **Edit** — plain language; one idea; moderate A; **relevant** examples; mark **important** phrases for bold.  
5. **Self-check** — guide § Self-check + examples § F + html-deck checklist.  
6. **Build HTML deck** — themed sections; every card Q+A visible; **Copy on every card**; bold important text; path under `~/explanations/`.  
7. **Handoff** — absolute path, card count, spine (not a full dump of all cards in chat).

## Card types (emit a mix)

| Type | Front cue | Back must include |
|------|-----------|-------------------|
| **First-principles / why** | Why X? What fails without X? | Need → constraint → shape |
| **Intuition** | Picture / analogy / “how does this work?” | Portable model + where analogy breaks if needed |
| **Contrast** | X vs Y | Decision table or sharp difference |
| **When to use** | When X? | Triggers + one “not when” |
| **Mechanism / code** | Elaborate snippet / what happens when… | Step-through + why |
| **Process** | First step / order of design | Principle of *order*, not tool spam |
| **Definition spine** | What is X? | Only if load-bearing; still add why/misuse |

## Front / back rules (always)

**Front** — one clear question (contrast, when, why, what happens, elaborate). Not “discuss X.”

**Back**

1. Lead sentence (spine) — bold the key claim.  
2. Why / constraint (short).  
3. Bullets for facets.  
4. Domain-true example when useful.  
5. Optional “not when” / trap.

**Bold (page + copy payload):** type names, critical outcomes, constraints, contrast poles — not whole paragraphs. See `html-deck.md`.

**Length:** one focused screen. **Atomic:** one *idea* per card.

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
| **Copy** | **Every** card has a Copy button; clipboard text keeps newlines, blanks, `-` bullets, and `**bold**` for important phrases |
| **Bold** | `<strong>` on important terms in the page |
| Self-contained | Inline CSS/JS only |
| No flip UI | Unless the user explicitly asks |

Implementation pattern and CSS: **`references/html-deck.md`** (load when writing the file).

**Chat-only markdown** only if the user says so; still keep the same craft rules. Prefer HTML even then if they want copy buttons.

## Do (quality bar)

- Understand → then cardize.  
- Prefer **why / intuition / contrast / when** over definition spam.  
- Plain language; moderate meaningful backs.  
- Domain-true examples.  
- HTML page with **copy on every card** and **bold** on important text.  
- Split independent ideas; keep why on the claim.

## Do not

- Deck of one-line jargon definitions as the main product.  
- Skip the HTML page when building a real deck (unless chat-only).  
- Cards without a working Copy control.  
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
