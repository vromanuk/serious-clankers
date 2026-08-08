# Create Flashcards

## Intent

Produce study flashcard decks that encode **understanding** (first principles, intuition, why, contrasts, when-to-use) in **moderate, plain-language** cards — not bare fact trivia. **Default visualization:** self-contained HTML page with always-visible Q/A, **Copy on every card** (formatting preserved), and **bold** on important terms. Craft from SuperMemo/Anki formulation rules plus the user’s deck style (systems, Rust, DDD).

## Triggers

- **SHOULD** apply when the user asks for flashcards, Anki/Quizlet cards, a study deck, spaced-repetition cards, or to turn notes/a chapter/a talk/an explanation into review cards.  
- **SHOULD** apply when the user asks how to write good flashcards for a topic they are learning.  
- **SHOULD NOT** apply when the main ask is a long teaching HTML page only (`explain-topic`) unless they also want a deck.  
- **SHOULD NOT** apply for multi-lens code review (`skeptic`) or PR bodies (`pr-description`).

## Behaviors

### Behavior: First principles before cards

The agent SHALL apply a first-principles pass (need, constraints, mechanism, drop-if-smaller, wrong default) before drafting cards, and SHALL NOT cardize slogans it cannot ground. When understanding is missing, the agent SHALL research or recommend explaining first.

#### Scenario: User pastes jargon list

- **GIVEN** only term names with no source model  
- **WHEN** asked for flashcards  
- **THEN** the agent builds or fetches enough model to state why/when, or asks for source material — not empty definitions

### Behavior: Intuition-first deck mix

The agent SHALL prefer card fronts that are why / contrast / when / what-happens / elaborate-code / process-order, and SHALL keep pure definition cards sparse. Answers SHALL include why or constraints when the idea is a design choice.

#### Scenario: Smart-pointer chapter

- **GIVEN** material on Box, Rc, RefCell  
- **WHEN** producing a deck  
- **THEN** the deck includes contrast and when-to-use cards, not only “What is Box?”

### Behavior: Moderate meaningful answers

The agent SHALL write answers that are scannable (lead + bullets) and moderate length (enough for intuition; not a chapter), in plain language.

#### Scenario: Interior mutability

- **GIVEN** RefCell runtime vs compile-time borrowing  
- **WHEN** writing the answer  
- **THEN** the card states the tradeoff (when each wins), not only the type name

### Behavior: Relevant examples only

The agent SHALL use domain-true examples for the topic (e.g. concurrent requests for server concurrency; order aggregates for consistency) and SHALL NOT default to irrelevant toy stories when a real domain example fits.

#### Scenario: Concurrency card

- **GIVEN** the learner works on services  
- **WHEN** illustrating a race  
- **THEN** the example uses concurrent requests / shared state, not a random kitchen scene as the sole model

### Behavior: One idea per card

The agent SHALL keep one primary idea per card. Independent guarantees SHALL be split. Why-for-this-idea stays on the same card.

### Behavior: HTML deck with copy and bold

Unless the user requests chat-only or another export format, the agent SHALL write a self-contained HTML file under `~/explanations/YYYY-MM-DD-flashcards-<slug>.html`. Every card SHALL show Q and A at once, SHALL include a **Copy** control that puts plain text on the clipboard with newlines and bullets preserved, and SHALL render important terms in bold (`<strong>`), with matching emphasis in the copy payload (e.g. `**term**`). The agent SHALL load `references/html-deck.md` when building that page.

#### Scenario: Default deck request

- **GIVEN** the user asks for flashcards on a topic  
- **WHEN** the agent finishes  
- **THEN** an HTML file exists with multiple cards, each with a working Copy button, and important type/outcome words are bold on the page

#### Scenario: Copy formatting

- **GIVEN** a card answer with lead sentence and bullets  
- **WHEN** the user hits Copy  
- **THEN** the clipboard text includes blank lines and `-` bullets so paste into notes/Quizlet keeps structure

## Constraints

### Constraint: Load the guide

For non-trivial decks the agent MUST load `references/guide.md` and SHOULD load `references/examples.md` (SuperMemo map + worked good/bad cards). For HTML output the agent MUST load `references/html-deck.md`.

### Constraint: Output is a visual deck

The agent MUST deliver an HTML deck by default (or card-writing rules if that was the only ask), not a substitute full course essay, unless the user also asked for explanation. Chat-only markdown is allowed only when the user asks for it.

### Constraint: No fake understanding

The agent MUST NOT invent mechanisms or tradeoffs; mark assumptions or verify sources.

<!-- skillet-version: 1.7.0 -->
