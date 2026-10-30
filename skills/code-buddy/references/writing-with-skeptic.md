# Writing with skeptic

Skeptic is the only source of the rules. This file does not copy them. It says which skeptic stages to load before
writing each kind of piece, and how to run those stages on the piece before showing it, so the code follows skeptic
while it is written instead of being fixed in review.

Paths are relative to the skills folder (`~/.agents/skills`).

## Before writing a piece: load its stages

Load each stage's `SKILL.md` and the references it names for that topic.

| Writing | Load |
|---|---|
| Every piece | `skeptic-naming`, `skeptic-comments`, `skeptic-hard-rules` |
| Public face: job struct, methods, public data types | `skeptic-architecture` (`components.md`, `type-driven.md`, `rust-apis.md`), `design-components` (`guide.md`, `philosophy-of-design.md`), `skeptic-complexity`, `skeptic-solid` |
| Rules and decisions (no IO) | `skeptic-testability` (`pure-core.md`), `skeptic-complexity`, `skeptic-conventions` |
| IO parts: clients, wrappers, callbacks | `skeptic-architecture`, `skeptic-reliability` (remote calls, queues, retries, shutdown), `skeptic-security` (TLS, keys, untrusted input) |
| Async code: tasks, locks, channels, `select!`, cancellation | `skeptic-async` and the references it names |
| Settings, defaults, constants | `skeptic-comments` (`comments.md` § Settings and numbers), `skeptic-naming` (public API), `skeptic-reliability` (limits, timeouts) |
| Errors | `skeptic-complexity` (error design), `skeptic-architecture` (`rust-apis.md`) |
| Metrics, logs, spans | `skeptic-observability` (and the `observability` skill it loads) |
| Tests | `skeptic-unit-tests` (and the `unit-tests` skill), `skeptic-testability` (`testing.md`) |

Before the first piece of a session, read the Scars list in `skeptic/SKILL.md`: the misses skeptic sees most often.

## While writing

Use the loaded stages' rules for the code being written, not as a check afterwards. When a stage's rule and the
brief disagree, stop and ask: that is a decision trigger.

## Before showing the piece: run its stages

1. Go through the stages in skeptic's order (purpose, architecture, complexity, testability, unit tests,
   observability, comments, naming, conventions, hard rules, then async and SOLID when they apply).
2. For each stage, ask its question from its `SKILL.md` about this piece only, using its references. Skip a stage
   only when it cannot apply, and say so in one line.
3. Fix what fails.
4. In the stop, under Checks, list the stages run, what was fixed, and anything left as an open decision.

This is not a full skeptic report. It covers only the piece in this stop.

## When the closing review finds something

The closing skeptic pass on the whole change should find little. For each finding, say which of these it is, and
propose the fix to the user:

- **A stage was not loaded or not run** for that kind of piece: fix the table above.
- **The stage has no rule for it:** propose a change to that skeptic stage.
