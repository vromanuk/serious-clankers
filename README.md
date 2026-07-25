# serious-clankers

Personal setup for coding agents.

## Layout

```text
skills/<name>/     # one folder per job
context/           # always-on preferences (AGENTS.md, …)
```

## Skill format

Skills use [Skillet](https://skillet.sentry.dev) for authoring and checks.

```text
skills/<name>/
  SKILL.md         # runtime instructions
  spec.md          # Skillet behavior contract
  SOURCES.md       # optional: provenance and decisions
  references/      # extra docs for this skill (load on demand)
  scripts/         # optional helpers
```

Skeptic keeps shared review depth under `skills/skeptic/references/` (including `examples/` for worked samples). Stages point there; they do not use a pack-level `references/` folder.
