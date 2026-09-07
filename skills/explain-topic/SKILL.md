---
name: explain-topic
description: >
  Create a rich, self-contained HTML explanation of a concept, technology, or
  pattern. Use when the user asks to explain, teach, or understand something —
  e.g. "explain how X works", "what is Y", "teach me about Z", "why does X
  exist?", "how does X compare to Y?", or any deeper mental-model ask. Also use
  when asked for "first-thinking principles", "first principles", intuition,
  high-level ideas, or a mental model of something — those all mean this skill.
  Produces a dated HTML file under ~/explanations/. First-thinking principles,
  why it exists, and intuition (with simple HTML figures) before detail.
---

# Explain Topic

Produce a single long-form HTML page that teaches a concept, technology, or
pattern. The page should make sense to a beginner while still giving an
experienced reader a clear path to the real mechanism.

The job is a **mental model**, not an API dump and not an implementation.

## Workflow

1. **Name the real question** in plain words. If the topic or depth is ambiguous,
   ask one short clarifying question (or state a clear assumption on the page).
2. **Think from first-thinking principles before writing HTML** (see below). Build a
   narrative: what need exists, what goes wrong without this idea, the smallest
   useful mental model, how the mechanism meets the need, what people confuse.
3. **Explore enough context** to be accurate: docs, surrounding code, a talk
   transcript, standards — prefer checked sources over speculation. Mark guesses.
4. **Write one self-contained HTML file** with inline CSS and JavaScript (JS only
   where interaction is needed — e.g. quiz). No external fonts, CDNs, images, or
   packages. Save under `~/explanations/YYYY-MM-DD-explanation-<slug>.html`
   using today’s local date so files sort by time and stay out of repos.
5. **Validate before handoff:** read *only the first paragraph of each section,
   in order* — it must tell a coherent story on its own (§ *Intuition before
   mechanism*); no section opens with syntax, config, a file layout or code;
   complete HTML document; no external deps; every code block has
   `white-space: pre` or `pre-wrap`; quiz works if present; JS has no parse
   errors; open the file when practical.

**Tiny asks only:** a one-notion trivia question may stay in chat. Offer a page
if they want depth. If they say chat-only, still use the same teaching order
without forcing a file.

## First-thinking principles (how to think) — applies *everywhere*

Asked for “first-thinking principles”, “first principles”, “the intuition”, “the
high-level idea”, or “why is it like this” — all the same request, and all mean
this section plus § *Intuition before mechanism*.

Prefer this over habit, “best practice,” and pattern-matching.
This is not only the “Why it exists” section. **Every** major claim, figure,
signature, and example on the page must survive this check.

For each idea, make explicit (in prose or a short callout):

1. **Need** — what job fails without this?
2. **Constraints** — what you cannot assume (no GC, many threads, caller still
   needs the value, OS has one close, …)?
3. **Mechanism** — smallest rule or type shape that meets the need under those
   constraints.
4. **What you would drop** if the need were smaller (e.g. if the caller never
   reused the string, you would move it, not borrow).

Also:

- Derive from fundamentals. Do not copy the usual blog structure without checking it fits.
- Restate the **real need** in plain terms. Strip inherited assumptions.
- Prefer **why** and **mechanism** over slogans. If a line does not change understanding, drop it.
- Verify analogies; say where they break. Do not force a stock metaphor cast.
- Fail bar: if a caption could be swapped for “best practice” with no loss of
  meaning, rewrite it until the need is visible.

### Separate what is forced from what was chosen

The most useful thing an explanation can do is say which parts of a design *had*
to be that way and which were judgement calls.

- **Forced** — follows necessarily from a constraint. *“Once values are
  compressed they are variable length, so a position can no longer be computed —
  an index becomes unavoidable.”*
- **Chosen** — defensible, could have gone the other way; usually about scope,
  risk or cost. *“It could average old data during a merge. It deliberately
  doesn’t, to stay small and dependable.”*

Blur the two and arbitrary decisions look like laws of nature, which hides where
the reader’s own situation might justify a different answer.

### Follow the consequence chain from one root fact

Where several traits share a cause, show the chain instead of listing the traits:

> series are unpredictable and short-lived → nothing can be reserved in advance →
> data must be written after the fact → each batch carries its own index →
> expiry becomes deleting a whole batch → churn costs nothing

One fact, five consequences. That reads as understanding; a feature list doesn’t.

**Shared-root test:** if one structural change fixes several problems that looked
unrelated, say so — it is strong evidence the diagnosis is right, and far more
convincing than three separate fixes.

### Name the assumption that could be false

Every mechanism rests on something being true about the data or the world. State
it, and state when it stops holding.

> This compression assumes monitoring data is repetitive — fixed intervals,
> values that rarely move. On genuinely random data it barely compresses at all.

**Fail bar:** a mechanism presented with no conditions under which it fails is
being sold, not explained.

### Mark the status of every number

When describing real systems keep three things visibly apart: **fixed by the
format** (state it plainly), **measured and published** (cite it), and **vendor
benchmark or your own inference** (label it as such, in place — not only in the
footer). A project’s performance claims about its own product are direction, not
measurement. In the footer, also separate what you verified from what you recalled.

## Intuition before mechanism — the most common failure

This skill fails most often by delivering **correct detail too early**: syntax,
config, file layouts, byte diagrams or numbered steps arriving before the reader
knows what the thing *is*. The detail is accurate and reads as trivia. Three
mechanisms prevent it.

### 1. Every section leads with the model, not the mechanism

“Whole picture before parts” applies **recursively — to every section**, not once
per page. It is easy to satisfy it in the Intuition section and then open each
later section with a config file. Don’t.

Order inside any section that explains something:

1. **What it is** — plain words, anchored on something the reader already uses.
2. **Why it must be that shape** — the constraint that forces it.
3. **What breaks without it** — the failure it exists to prevent.
4. **How it works** — mechanism, syntax, layout, steps, bytes.
5. **What it costs** — the honest trade, so it reads as engineering, not a pitch.

For a page covering several systems or components, add an **orientation block
before the deep dives**: each one in two or three sentences, so the reader has a
map before any of them is opened up.

**Fail bar:** read *only the first paragraph of each section, in order*. If that
alone doesn’t tell a coherent story, the models are buried inside the mechanisms.

### 2. Derive it — don’t announce it

When a design would look arbitrary if simply stated, **start from the naive thing
the reader would do and fix one problem at a time** until the real design falls
out. Then name what was built: “you have just invented X.”

Each step says: what we gained, what is still broken, what that forces next. The
target feeling is “I’d have got here myself,” not “I have been told this.”

This also produces the strongest first-principles writing available, because the
constraint is *demonstrated* before the conclusion instead of asserted after it.

**Fail bar:** if the page says “X does A, B and C” without ever showing what goes
wrong when you skip A, the reader has facts and no model.

### 3. One familiar anchor per part, not one per page

Each major component earns its own everyday instance. Do not stretch a single
metaphor across the whole page, and do not build an invented world (see
Examples § *Analogies*). A reader dropping into any section cold should meet
something recognisable within two sentences.

**Fail bar:** a section whose opening only makes sense if you read the one before.

## Required page structure

One continuous page (no top-level tabs). Title, short summary (the spine in one
or two sentences), table of contents, then these sections **in order**:

1. **Background**  
   Orient the reader. We do not know how much they already know.  
   - **Deep background** (skippable): enough surrounding context to stand the idea up from zero. Label it so a familiar reader can skip.  
   - **Narrow background**: only what this topic needs next (where it sits in a larger system, related ideas, vocabulary used later).

2. **Why it exists**  
   The real need and the failure mode without it. Concrete situation, not abstract fear.  
   End with the first-thinking-principles link: constraints → why this shape of
   solution appears, and which parts of it were forced versus chosen.

3. **Intuition**  
   The **core idea** before full detail. Essence only.  
   - Small **toy examples** that are **familiar, simple, and teaching-first**
     (see Examples).  
   - **Anchor on a real, familiar instance of the topic — not an invented
     metaphor.** A technical topic almost always has established, everyday
     instances the reader already knows (CSV fields / a filesystem path for
     string splitting, a stack of plates for a stack, a phone book for a hash
     map). Build intuition on one of those. Only invent a fresh metaphor
     (“imagine a tape with a probe sliding along it…”) when no familiar instance
     of the actual topic exists — see Examples § *Analogies*.  
   - **Figures and diagrams liberally** (see Diagrams).  
   - Name the parts and the order of events in plain words.  
   High-level idea first; mechanism detail after — not the reverse.

   **Whole picture before parts** — and see § *Intuition before mechanism*, which
   applies this to **every** section, not only this one. For any multi-step mechanism (lifecycle,
   protocol, API), first show **one figure or paragraph that is the entire
   story** (what the thing *is*, what is owned, when cleanup runs, how it
   differs from the usual alternative). Only then zoom into numbered steps or
   component details. Do not open with a three-box pipeline that the reader
   must assemble into a model themselves.

4. **How it works**  
   Build up: simple case → one complication → only needed edge cases.  
   Derive the mechanism from the need. Introduce APIs, type names, and jargon
   only after the model lands; define jargon on first use.
   Detail sections should **refer back** to the whole-picture figure (“this is
   moment 2 inside that story”), not introduce a disconnected second metaphor.

5. **Compare** (include when it helps)  
   What neighboring options optimize for; when you would pick which; honest trade-offs.

6. **Quiz**  
   Exactly **five** medium-difficulty interactive multiple-choice questions that
   test whether the reader understood the substance (see Quiz quality).

7. **Study cards** (optional, dense multi-idea topics)  
   Separate study aid — **not** a dump of the page’s diagram captions.
   See **Study cards / flashcards** below. Always-visible Q/A (no flip UI).
   For a **standalone deck** (Quizlet/Anki export, many cards, full craft): skill
   **`create-flashcards`**. Do not mix figure markup or labeled template fields
   (SCENE / REMEMBER / …) into answers.

Smooth transitions between sections. Engaging, clear prose (systems writing with
flow — think careful technical essays, not slide bullets). Plain words; common
word first.

## Examples (every one counts)

Whenever you show an example — in prose, a figure, a callout, or a code block —
it must be **relevant, simple, teaching, and familiar**. Not a decorative snippet.

### Prefer canonical teaching sources first

When a **famous, standard teaching text** already has a classic example for this
idea, **use that example** (names, shape, and story), lightly attributed — do not
invent a weaker parallel.

| Domain | Prefer first |
|---|---|
| Rust language basics | *[The Rust Programming Language](https://doc.rust-lang.org/book/)* (e.g. ownership: `String::from("hello")` + `takes_ownership`; borrowing: `calculate_length(&s1)`) |
| Rust by small programs | *Rust By Example* when it has the standard demo |
| Algorithms / systems ideas | CLRS, *Designing Data-Intensive Applications*, OS texts — only the well-known figures |
| Language X | That language’s official book / Tour / effective guide |

Use the canonical cast and code shape; do **not** litter the page with meta labels
like “(Book-style …)”, “Rust Book:”, or “Familiar Book demo.” A single quiet
source link is enough when attribution matters.

### When no canonical example fits

Then invent one that is still familiar and teaching-first:

- Situations the reader knows: checkout + email, photo upload, config file, chat
  message, button counter, open/save document.
- Everyday language first, then a tiny program that mirrors that story.
- Beginner-facing APIs (`String`, `Vec`, `File`, `Mutex`) over pseudo-C
  (`allocate` / `free` / `buf`) unless allocators *are* the topic.
- One idea per example; same cast across before/after.

**Do not**

- Replace a standard Book example with a vague custom one “for originality.”
- Abstract labels alone (`Component A` / `B`) when a named role or Book cast works.
- Obscure domain objects or enterprise noise that hides the idea.
- Examples that only make sense after you already know the concept.

Fail bar: if a sharp beginner asks “who is A and what is buf?”, rewrite.
If the Book already has `takes_ownership(s)` and you wrote `takes(s)` with `"hi"`,
use the Book version instead.

### Analogies / metaphors (last resort, not first reach)

An invented metaphor (“picture a tape with a probe”, “think of it as a
conveyor belt”) is a **fallback**, not the default way to build intuition. It
adds a second thing the reader must learn and map back. Reach for the topic’s
own **familiar instances** first.

Order of preference for the Intuition anchor:

1. **A real, familiar instance of the topic itself.** Most technical topics
   have one the reader already lives with. Use it with real values.
   - String split → a **CSV/TSV row** (`"Ada,Lovelace,1815".split(",")`), or a
     **path** (`"/usr/local/bin".split("/")`, `PATH.split(":")`).
   - Stack → a stack of plates / browser back button. Queue → a checkout line.
   - Hash map → a phone book / coat-check tickets. Cache → a desk vs a filing
     cabinet you already reach into.
   - Retries/backoff → redialing a busy phone number.
2. **The canonical Book/teaching example** (see the table above), if the topic
   is language/algorithm lore with a standard demo.
3. **An invented metaphor** — only when neither exists, and only if it maps
   cleanly. If you invent one, say where the analogy breaks (per Diagrams
   rules), and still show the real mechanism right after.

**Do not** open with a made-up metaphor when a familiar instance of the actual
topic is one sentence away. A metaphor that restates the mechanism in fictional
props (cells, probes, tapes, widgets) usually teaches less than the real thing
with real values.

Fail bar: if the metaphor could be deleted and replaced by “here is a real
`X` you already use” with **no loss** of understanding, delete it and use the
real `X`. If a reader would learn the prop (the tape) instead of the topic
(split), rewrite.

## Diagrams

Pick a **small number of diagram families** and reuse them across the page so
cases compare cleanly. Figures support intuition; they are not a product feature.
Every diagram that moves data or ownership must use the **Examples** bar above.

### Prefer these families (high-level overview style)

- **Before / after** panels (without the idea vs with it, or old vs new behavior).
- **System / flow** diagrams: data, control, or ownership between parts — always
  include **example values** on the arrows or boxes.
- **Simplified UI sketch** only when the topic is user-facing (optional).
- **Component / boundary** cards: who owns what, who calls whom.
- **Compact tables** for mappings, invariants, toy inputs → outputs.
- **Numbered static steps** as a vertical or horizontal list of fully visible
  cards (1 → 2 → 3), all readable at once.

### Rules

- **Never use ASCII diagrams** as the main figure.
- **Always use simple HTML + CSS** for diagrams (cards, flex/grid, panels, lists).
  Not screenshots of tools. Not Mermaid unless the user asks.
- **Default is static and fully visible.** The reader should understand the figure
  by looking, without pressing Play, waiting for animation, or hunting which box
  is “lit.” Numbered steps and before/after panels beat step-reveal UIs.
- **Do not build Play / Next / chip “tour” diagram players** unless the user
  explicitly asks for interactive walkthroughs. They often add complexity without
  teaching better, and they break easily.
- If you use any JS for diagrams at all, keep it tiny and test it; quiz JS is the
  main interactivity this skill expects.
- Clarity: plain title + short subtitle on every box; example data when something
  moves; one main idea per figure; split overloaded posters.
- Use **callouts** for key definitions, invariants, edge cases, traps.

More tokens and panel CSS: `references/html-visual-kit.md` (load when building).

## Quiz quality

Treat the quiz as part of teaching, not decoration. Inspect all five questions
as a set before shipping.

- Medium difficulty: need the substance of the page, not a single copied phrase.
- No gotchas, jokes, “all/none of the above,” or impossible options.
- Every distractor is a **plausible misunderstanding**.
- **Randomize option order** per question (deterministic seed OK). Do not park the
  correct answer always first/second. Balance correct positions across the five.
- Keep options **comparable** in length, grammar, and confidence. Do not make the
  correct option the longest or most precise by default.
- Clicking an option shows immediately whether it is correct and **why** (and,
  when useful, the misconception behind a wrong pick).
- Feedback lives in the page’s own JS/DOM (offline). Do not expose correctness
  before click via styling, order, or labels.

## Study cards / flashcards (optional — separate from visuals)

**Separate teaching media.** The long page (and its figures) teach by reading.
Study cards are for **building and checking intuition**, then for quick review
and **easy copy-out**: plain **question + answer**, both always visible.

Primary job is **not** a glossary of definitions. Prefer cards that make the
reader *picture a mechanism* or *trace a familiar value through the design*.

### Presentation (required)

- **Static list — no flip, no hide-answer, no click-to-reveal.**
- Each card shows both sides at once in a copy-friendly shape, e.g.:

  ```text
  Q: How can the string "hello" be implemented internally?
  A: …concrete layout walkthrough across the designs on the page…
  ```

- Style Q and A differently if you like (weight, background, left border) so
  they scan well — but keep them **readable and selectable** in one glance.
- **Every card gets a Copy control** that puts the full card on the clipboard as
  plain text with formatting preserved (newlines and bullets), e.g.:

  ```text
  Q: …

  A:
  Lead sentence.

  - bullet one

  - bullet two
  ```

  Use `navigator.clipboard.writeText` with a `textarea`/`execCommand` fallback.
  Brief “Copied” feedback on the button is enough; do not require selecting text.
- Optional topic tags/filters are fine; filters must not hide the answer mode
  behind interaction beyond showing/hiding whole cards.
- Do **not** build Quizlet-style flip cards, 3D transforms, or “click to see
  answer” unless the user explicitly asks for that.

### Prefer intuition cards (general pattern)

**Best cards start from a tiny, familiar concrete value or event**, then ask
how the topic’s mechanism represents it or what happens next. The answer walks
the reader through memory, ownership, control flow, or tradeoffs — not a
one-line slogan.

Generalize this shape (topic-agnostic):

| Pattern | Question shape | What the answer does |
|---|---|---|
| **Toy value through the layout** | “How can *X* be stored / represented internally?” | Same familiar value under each design on the page (e.g. `"hello"`, empty list, one connection) |
| **One operation** | “What happens when you *copy / move / free / append* *X*?” | Step the mechanism; name what is shared vs duplicated |
| **Boundary case** | “What changes if *X* is empty / huge / concurrent?” | Show which fields or paths kick in |
| **Contrast** | “Why does design A do *this* for *X* while B does *that*?” | Constraints → different answer for the same toy |

Canonical examples of the pattern (style, not mandatory wording):

- Strings: *How can `"hello"` be implemented internally?* → SSO inline vs heap
  vs COW shared buffer vs German 16-byte header.
- Ownership: *What happens to the heap buffer when `s` is moved?*
- Async: *What does the state machine look like while awaiting `read`?*
- HTTP: *Where do the bytes of a 200 response body live as the client reads?*

**Include at least one toy-value / one-operation card** when the topic is a
representation, lifecycle, or multi-design comparison. That card should be
early in the deck when it captures the page’s spine.

### What a good card looks like

**Q:** prefer the intuition patterns above; also fine: “Elaborate on this
code: …” / a sharp “What is X?” only when X is easy to misuse.

**A:** readable, scannable prose — **not one dense paragraph**. Structure:

1. **Lead sentence** — the spine of the answer in one line.
2. **Bullets** when listing designs, steps, downsides, or alternatives (one idea
   per bullet; blank line between bullets is fine for copy-paste comfort).
3. **Closing line** only if it ties the list back (optional).

Include as needed:

- why the answer makes sense (first principles, briefly),
- the **concrete walkthrough** (where bytes live, who owns them, what grows),
- a short code sample or error message when useful,
- a concrete fix or consequence,

…written as normal text + `-` bullets, not as `SCENE:` / `RULE:` / `REMEMBER:` /
`TRAP:` labels. In HTML, preserve newlines (`white-space: pre-wrap` on the
answer body) so the list stays readable and easy to copy.

**Good (scannable):**

```text
Q: How can the string "hello" be implemented internally?
A: Several different layouts can represent the same five characters.

- C: a pointer to H e l l o \0 in some buffer you manage (no length field).

- Modern C++ with SSO: object holds length 5; characters live in the inline
  buffer — no heap (5 is under the small-string limit).

- Rust String: pointer + length + capacity; five bytes on the heap (no SSO
  in std). &str is pointer + length only.

- German string: length ≤ 12 → all five characters inline in the 16-byte value.

- COW (historical): one shared heap buffer + refcount until a write.

Same text, different answers to where / how long / may I grow.
```

**Bad (hard to read / hard to copy):** the same content as one long run-on
paragraph with “C: … Modern C++: … Rust: …” jammed together.

Weaker cards (avoid as the main deck): pure term → definition with no picture
of a value or step (“What is SSO?” with only “small string optimization”).
Better: *Where do the characters of `"ok"` live with SSO vs without?*

### Separation rules

| Lives on the **page** | Lives on a **study card** |
|---|---|
| Before/after figures, flow diagrams, tables | Short Q + prose answer |
| Callouts, long walkthroughs | Optional small code block inside the answer |
| Whole-picture diagrams | No diagram shells, no Play/flip UI, no “caption for figure 3” |

- Do **not** put the page’s visual structure into the answer.
- Do **not** use template headings (Scene, Mental model, Remember, Trap) on cards.
- First principles still apply: the answer should say *why*, not only a slogan.
- Prefer one clear idea per card; many cards beat one mega-card.
- When comparing several designs, one **toy value across designs** card beats
  five isolated jargon cards.

### When to include study cards

Dense multi-idea topics (e.g. a whole talk or multi-layout comparison). Skip
for a single short concept if the quiz is enough.

## HTML and code-block constraints

- Self-contained: inline CSS and JS only.
- Escape content for HTML/JS. Preserve meaningful whitespace in code.
- Code: `<pre><code>…</code></pre>`. CSS for `pre` must set `white-space: pre`
  or `pre-wrap`. Scan the saved source before delivery.
- Responsive enough to read on a phone.
- Visible focus states; do not use color alone for correctness.
- Small, dependency-free JS. Fix parse errors before handoff (unescaped `"` in
  strings is a common break).
- Distinguish observed fact from interpretation.

## Output location

```text
~/explanations/YYYY-MM-DD-explanation-<short-slug>.html
```

Create `~/explanations/` if needed. Do not put the file inside a code repo unless
the user asks.

## Final handoff (chat)

- Absolute path to the HTML file (openable locally).
- One-sentence spine of the explanation.
- What you based it on (docs, code, talk, …) and any assumptions.
- Do not paste the whole page into chat.

## Do not

- Lead with API catalogs, flag lists, or jargon maps.
- Open any section with syntax, config, a file layout, byte diagrams or code
  before the reader knows what the thing is and why it exists.
- Assert a design you could have derived. If it looks arbitrary stated flat,
  build it up from the naive version instead.
- Satisfy “whole picture first” once at the top and then dive straight into
  mechanism in every section after it.
- Recite “best practice” without when it helps and when it fails.
- Present a judgement call as if it were forced by physics, or a hard constraint
  as if it were a preference.
- Repeat a vendor’s benchmark as though it were a measured property of the world.
- Implement a full solution when they asked to understand.
- Use formal, padded language or empty slogans.
- Default to animated / Play-based diagram players.
- Default to flip / click-to-reveal study cards (use always-visible Q/A).
- Turn “how does this **file** work?” into a pure concept lecture (stay on the
  code if that is the ask).
- Treat “just write the code” as this skill.

## Scope

**In:** concepts, technologies, patterns, designs — including learning from a
talk or article when they want a durable model.

**Out:** main goal is implement/refactor; pure pair-programming with no teach
ask; PR/diff walkthroughs (use a diff-explanation skill if available).

## Progressive disclosure

| File | When |
|---|---|
| This `SKILL.md` | Always when the skill applies |
| `references/html-visual-kit.md` | Building diagrams, tokens, quiz/card UI details |
| `spec.md` | Contract and scenario checks |
