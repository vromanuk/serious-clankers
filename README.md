# serious-clankers

Personal setup for coding agents.

## Layout

```text
context/           # always-on (AGENTS.md)
skills/<name>/     # one folder per job
```

## Skill format

[Skillet](https://skillet.sentry.dev) skill layout:

```text
skills/<name>/
  SKILL.md              # runtime instructions
  spec.md               # behavior contract
  evals/cases/          # optional mechanical checks
  references/           # optional load-on-demand depth
```

Only add `references/` or `evals/` when the skill needs them. No pack-level shared library — each skill keeps its own depth.

Skeptic stages follow that rule: e.g. hard-rules live under `skeptic-hard-rules/references/`, not under the coordinator.

**Default project architecture** (always-on in `context/AGENTS.md`): job-shaped components, not app-wide layers; public face = **one job struct + use-case methods** that orchestrate once; interior = pure-core/shell (sans-IO); deep-module judgment from Ousterhout under `design-components/references/philosophy-of-design.md`. Layout: `skills/skeptic-architecture/references/components.md`. Design workflow: skill **`design-components`**. Study decks: skill **`create-flashcards`** (intuition/why first, moderate cards).
