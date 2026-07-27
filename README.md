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

**Default project architecture** (always-on in `context/AGENTS.md`): job-shaped components, not layers. Depth: `skills/skeptic-architecture/references/components.md`.
