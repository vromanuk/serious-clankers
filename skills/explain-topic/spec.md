# Explain Topic

## Intent

Explain a concept, technology, or pattern so the user builds a solid mental model. Teaching uses first principles: derive from fundamentals and real need, not habit or slogans. Answers give background (deep then narrow), lead with intuition and concrete examples, then add detail only as needed. Teaching is the job — not shipping an implementation.

## Triggers

- **SHOULD** apply when the user asks to explain, teach, or understand a concept, technology, or pattern.
- **SHOULD** apply when the user wants a deeper model of an idea even if they do not say “explain”.
- **SHOULD NOT** apply when the main ask is a line-by-line walkthrough of a specific source file (explain the *idea* if useful, but do not treat that as this skill’s full job).
- **SHOULD NOT** apply when the primary goal is shipping code or running a task with no teaching ask.
- **SHOULD NOT** apply when the user only wants hands-on pair implementation without an explain/understand ask.

## Behaviors

### Behavior: Restate the question

The agent SHALL restate the real question in plain words, and SHALL ask a short clarifying question when the topic or depth is ambiguous instead of guessing silently.

#### Scenario: Ambiguous depth

- **GIVEN** the user says "tell me about Kafka"
- **WHEN** depth is unclear
- **THEN** the agent names likely angles in plain words and asks which depth they want, or starts with a beginner model and offers to go deeper

### Behavior: Background deep then narrow

The agent SHALL give deep background for beginners (marked skippable if the reader already knows it) and then narrow background directly relevant to the question, exploring surrounding context before focusing.

#### Scenario: Explaining a pattern in a larger system

- **GIVEN** the user asks how backpressure fits in their pipeline
- **WHEN** the agent explains
- **THEN** the answer first orients a beginner to the surrounding flow (skippable if known), then narrows to the part that answers the question

### Behavior: Intuition before detail

The agent SHALL lead with the core intuition (essence, not full detail), a concrete toy example, and diagrams when structure or flow helps, and SHALL put deeper mechanism after that.

#### Scenario: User asks how backpressure works

- **GIVEN** the user asks "how does backpressure work?"
- **WHEN** the agent explains
- **THEN** the answer starts with the problem, a small example, and a simple diagram if useful, before protocol or API specifics

### Behavior: First principles

The agent SHALL treat first principles as the primary teaching method: derive from fundamentals and the real need; restate the question in plain terms; prefer why and mechanisms over slogans; verify analogies; and drop content that does not change understanding. The agent SHALL NOT rely on “best practice” labels without saying when they apply and when they fail.

#### Scenario: User asks why immutability is used

- **GIVEN** the user asks "why do people use immutable data?"
- **WHEN** the agent explains
- **THEN** the answer starts from the problem and the mechanism, not a list of trendy rules

### Behavior: Layered depth

The agent SHALL build from the simple case to complications, match length to the ask, and offer a clear path to go deeper instead of dumping every edge case up front.

#### Scenario: User wants a short comparison

- **GIVEN** the user asks "what's the difference between threads and async briefly?"
- **WHEN** the agent explains
- **THEN** the answer stays short, contrasts what each optimizes for, and offers deeper follow-ups

### Behavior: Understanding check

The agent SHALL end with a brief check that the idea landed and invite a focused follow-up.

#### Scenario: Finished explanation

- **GIVEN** the agent has explained the main model
- **WHEN** the explanation closes
- **THEN** it includes a short "you should be able to …" check or one question that tests the core idea

## Constraints

### Constraint: Teaching not shipping

The agent MUST NOT treat an explain-topic request as a request to implement a full solution unless the user also clearly asks for that work.

### Constraint: Plain language

The agent MUST use plain words, avoid corporate or padded tone, and separate facts from guesses.

<!-- skillet-version: 1.7.0 -->
