# Writing with skeptic

Skeptic is the only source of the rules. This file does not copy them. It says which skeptic stages to load
**before presenting** each kind of piece (an API, a design option, code), how to use them while comparing options
and writing, and how to run them on the piece before showing it. So the code follows skeptic while it is written
instead of being fixed in review.

Paths are relative to the skills folder (`~/.agents/skills`).

## Before presenting a piece: load its stages

Load each stage's `SKILL.md` and the references it names for that topic, then list them under "Skeptic rules that
apply" (the question) or "Skeptic rules used" (the code step).

| Presenting | Load |
|---|---|
| Every piece | `skeptic-naming`, `skeptic-comments`, `skeptic-hard-rules` |
| Behavior notes: edge cases and their handling | `skeptic-purpose` (real need), `skeptic-reliability` (calls to outside systems, retries, restarts), `skeptic-testability` (`pure-core.md`: decisions as data) |
| Public API: job struct, methods, public data types | `skeptic-architecture` (`components.md`, `component-api.md`, `type-driven.md`, `rust-apis.md`), `design-components` (`guide.md`, `philosophy-of-design.md`), `skeptic-complexity`, `skeptic-solid` |
| Decision code (no IO) | `skeptic-testability` (`pure-core.md`, `testing.md`), `skeptic-architecture` (`type-driven.md`: enum of states or typestate), `skeptic-complexity`, `skeptic-conventions` |
| IO parts: clients, wrappers, callbacks | `skeptic-architecture`, `skeptic-reliability` (remote calls, queues, retries, shutdown), `skeptic-security` (TLS, keys, untrusted input) |
| Async code: tasks, locks, channels, `select!`, cancellation | `skeptic-async` and the references it names |
| Settings, defaults, constants | `skeptic-comments` (`comments.md` § Settings and numbers), `skeptic-naming` (public API), `skeptic-reliability` (limits, timeouts) |
| Errors | `skeptic-complexity` (error design), `skeptic-architecture` (`rust-apis.md`) |
| Metrics, logs, spans | `skeptic-observability` (and the `observability` skill it loads) |
| Tests | `skeptic-unit-tests` (and the `unit-tests` skill), `skeptic-testability` (`testing.md`); one test for each agreed Given/When/Then case in `behavior.md`; when the crate has a `spec/SPEC.md`, the project's `spec-driven-development` skill, and one test for each acceptance criterion the change adds or changes |

Before the first piece of a session, read the Scars list in `skeptic/SKILL.md`: the misses skeptic sees most often.

## While comparing options

For each option in a question, say which loaded rule it meets or breaks. An option that breaks a rule can still be
recommended, but say which rule, why, and what it costs. The trade-off rests on the stages, not on taste.

## While writing

Use the loaded stages' rules for the code being written, not as a check afterwards. When a stage's rule and the
build plan disagree, stop and ask: that is a decision trigger.

## Before showing the code: run its stages

1. Go through the stages in skeptic's order (purpose, architecture, complexity, testability, unit tests,
   observability, comments, naming, conventions, hard rules, then async and SOLID when they apply).
2. For each stage, ask its question from its `SKILL.md` about this piece only, using its references. Skip a stage
   only when it cannot apply, and say so in one line.
3. Fix what fails within the agreed design. A fix that changes the design is a decision: stop and ask.
4. In the code step, under Checks, list the stages run, what was fixed, and anything left as an open decision.

This is not a full skeptic report. It covers only the piece in this step.

## When the closing review finds something

The closing skeptic review on the whole change should find little. For each finding, say which of these it is,
and propose the fix to the user:

- **A stage was not loaded or not run** for that kind of piece: fix the table above.
- **The stage has no rule for it:** propose a change to that skeptic stage.
