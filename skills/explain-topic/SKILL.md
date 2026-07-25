---
name: explain-topic
description: >
  Explains a concept, technology, or pattern from first principles.
  Use when the user asks to explain, teach, or understand something — e.g.
  "explain how X works", "what is Y", "teach me about Z". Also use when the
  user asks "why does X exist?", "how does X compare to Y?", "what's the
  difference between X and Y?", "when would you use X?", or any variation of
  wanting to understand a concept, pattern, or technology at a deeper level —
  even if they don't explicitly say "explain".
---

# Explain topic

Teach a concept from first principles. The user should leave with a clear mental model, not a pile of jargon.

## First principles

**This is the main way to think and teach. Prefer it over habit, “best practice,” and pattern-matching.**

- Derive from fundamentals. Do not copy “how it’s usually explained” without checking it fits.
- Restate the real need or question in plain terms. Strip inherited assumptions.
- Verify analogies. Question “best practices” without context.
- Why over what/how. Prefer mechanisms over slogans — if a line does not change understanding, drop it.
- Checklist: name the need → real constraints → decided vs guessed → smallest mechanism → what you would drop if the need were smaller.

## Shape of the answer

### Background

We do not know how much the reader already knows.

1. **Deep background** (for beginners) — enough surrounding context to stand the idea up from zero. Mark it so a familiar reader can skip.
2. **Narrow background** — only what is directly relevant to *this* question.

Explore the real surroundings of the topic (related ideas, where it sits in a larger system) before narrowing.

### Intuition

Explain the **core intuition** — the essence, not the full detail.

- Concrete toy examples (small numbers, simple cases).
- Diagrams liberally when structure or flow helps (including component diagrams).
- High-level idea first; mechanism detail after.

## Do this

1. **Name the real question** in plain words. If ambiguous, ask one short clarifying question.
2. **Background** — deep (skippable) then narrow (see above).
3. **Intuition** — essence, toy example, diagrams (see above).
4. **Derive (first principles)** — need → mechanism. Use the section above; do not recite labels.
5. **Build up** — simple case → one complication → edge cases only if needed. Match depth to the ask.
6. **Compare when useful** — what each option optimizes for; when you’d pick which.
7. **Check understanding** — short “you should be able to …” or one question that tests the core idea.

## Voice

- Direct. Plain words. Common word first.
- Talk like a sharp coworker, not a textbook or HR doc.
- Analogies welcome when they help; say where they break.
- Separate fact from guess.

## Do not

- Dump a glossary or API reference unless they asked for that.
- Recite “best practice” without saying when it helps and when it fails.
- Silently implement a full solution when they asked to understand.
- Use formal, padded language.
- Turn a “how does this file work?” walkthrough into a pure concept lecture (stay on the code if that is the ask).
- Treat “just write the code” as an explain-topic job.

## Scope

This skill is for **concepts, technologies, and patterns** — mental models.

Not for:

- Implementing or refactoring as the main deliverable  
- Hands-on “build with me” practice when they want to drive the code themselves  
- Full-repo orientation maps (unless they asked how a *concept* fits)
