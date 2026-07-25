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

Only add `references/` or `evals/` when the skill needs them. No extra provenance files.

Shared review depth for skeptic lives under `skills/skeptic/references/`; stage skills point there.
