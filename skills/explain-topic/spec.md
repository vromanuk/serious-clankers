# Explain Topic

## Intent

Teach a concept, technology, or pattern so the user builds a solid mental model.
Thinking is first principles: real need, constraints, smallest mechanism — not slogans.
Default deliverable is a self-contained HTML page under `~/explanations/`, with
background, why, intuition (and **static** simple HTML figures) before mechanism
detail, plus a five-question quiz.

## Triggers

- **SHOULD** apply when the user asks to explain, teach, or understand a concept, technology, or pattern.
- **SHOULD** apply for deeper mental-model asks even without the word “explain”.
- **SHOULD NOT** apply when the main ask is implement/refactor with no teach ask.
- **SHOULD NOT** apply when the main ask is a PR/diff walkthrough (use a diff-explanation skill).

## Behaviors

### Behavior: Restate the question

The agent SHALL restate the real question in plain words and SHALL clarify depth when ambiguous.

### Behavior: First principles narrative

The agent SHALL derive the explanation from need and constraints before listing features or APIs. The agent SHALL apply need → constraints → mechanism (and “what you would drop if the need were smaller”) to every major section, figure, signature, and example — not only the “Why it exists” section. Slogan-only captions without a visible need SHALL be rewritten.

### Behavior: Required section order

The agent SHALL produce a single continuous HTML page with: Background (deep skippable, then narrow), Why it exists, Intuition, How it works, Compare when useful, Quiz (five interactive MC questions).

### Behavior: Static HTML diagrams

The agent SHALL use simple HTML/CSS diagram families (before/after, flow with example data, component cards, tables, fully visible numbered steps). The agent SHALL NOT default to Play/Next/chip diagram players. The agent SHALL NOT use ASCII as the primary figure.

### Behavior: Familiar teaching examples

The agent SHALL make every example relevant, simple, teaching-first, and familiar. When a famous standard text already has a classic example for the idea (e.g. The Rust Book for ownership/borrowing), the agent SHALL prefer that example over inventing a weaker custom one, and SHALL attribute it briefly. Otherwise the agent SHALL use familiar named roles/situations and beginner-facing APIs — not bare abstract labels with pseudo-allocator snippets.

### Behavior: Flashcards separate from visuals

When flashcards are included, the agent SHALL use a Quizlet-style format: question on the front, natural prose answer on the back (with optional short code). The agent SHALL NOT use labeled template fields (Scene, Remember, Trap, etc.) on card backs, and SHALL NOT embed the page’s diagram shells in cards. Figures belong only in the main article sections.

### Behavior: HTML artifact

For non-trivial topics the agent SHALL write `~/explanations/YYYY-MM-DD-explanation-<slug>.html` with inline CSS/JS only and return the path in chat.

### Behavior: Quiz quality

Five medium-difficulty MC questions; comparable option length; randomized order; immediate feedback with explanations.

### Behavior: Understanding not shipping

The agent SHALL NOT treat an explain-topic request as a full implementation request unless the user also asks for that.

### Behavior: Plain language

The agent SHALL use plain words, define jargon on first use, and separate fact from guess.

## Constraints

### Constraint: Self-contained HTML

No external CDNs, fonts, or JS packages for the default HTML deliverable.

### Constraint: No ASCII primary diagrams

MUST NOT use ASCII diagrams as the primary visual for non-trivial topics.

### Constraint: Code whitespace

Every code block’s CSS MUST use `white-space: pre` or `pre-wrap`.

<!-- skillet-version: 2.1.0 -->
