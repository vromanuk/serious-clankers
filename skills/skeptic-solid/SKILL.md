---
name: skeptic-solid
description: >
  SOLID and package design review (Rust): single responsibility as reasons to
  change, open/closed, substitutable trait implementations, client-specific
  traits, dependency direction and when a trait is worth adding; package
  cohesion (release/reuse, common reuse, common closure) and coupling
  (acyclic, stable dependencies, stable abstractions with instability and
  abstractness metrics); CUPID properties as the counterweight. Use when the
  user asks for a SOLID or package-design review, or when skeptic runs and the
  diff adds or changes traits, types with behavior, modules, or crates.
---

# Skeptic SOLID

**Question:** Will the next likely change be small and local — or does this design make it spread, break unrelated code, or force callers to work around it?

## When it runs

Run when the diff adds or changes traits, trait implementations, structs or enums with behavior, module layout, crate boundaries, `Cargo.toml` dependencies, or public API of a library crate. Otherwise omit the section (or, run alone, report `solid: none` with a one-line reason).

## Load

Load only what the diff touches:

1. `references/class-principles.md` — how to judge (change pressure, symptoms, pack defaults), then single responsibility, open/closed, substitution, interface segregation, dependency inversion, and when to add a trait at all  
2. `references/package-principles.md` — crates vs modules, release/reuse, common reuse, common closure and their tension, acyclic dependencies, stable dependencies (instability), stable abstractions (abstractness, main sequence)  
3. `references/cupid.md` — composable, does one thing well, predictable, idiomatic, domain-based; North's critique as checks on SOLID findings  

## Do

1. **Name the change pressure first.** For each candidate finding, state what kind of change it is about: which group of people asks for changes, which variant keeps being added, which caller is confused, which volatile dependency leaks in. No named pressure → no finding.  
2. **Single responsibility:** flag types and modules that answer to different groups (rules beside SQL beside formatting), the symptom set (many fields, many public methods, methods using disjoint fields, specific tasks in private helpers), and growing constructors. Do not flag code that changes together; check for artificial seams first.  
3. **Open/closed:** flag the same `match`/`if` on a kind repeated across functions or modules when the set keeps growing from outside, hard-coded construction of variants, factories with hard-coded lists, temporal coupling (`prepare()` before `run()`), and mutable shared services. A closed set the crate owns stays an enum.  
4. **Substitution:** flag trait implementations that panic, return unsupported, silently no-op, reject inputs the trait accepts, or return less than promised; callers that downcast or test `supports_x()`; "only works with …" wording; shared traits with no written contract. Prefer: add the method, split the trait, or remove the abstraction.  
5. **Interface segregation:** flag traits used in disjoint subsets by different kinds of client, configure-and-use on one interface, clients taking the concrete type, items made `pub` for one sibling, header traits that mirror one struct, and test fakes full of `unimplemented!()`. Split by kind of client, not per caller.  
6. **Dependency inversion:** flag policy depending on mechanism, dependencies named for the technology instead of the role, traits that leak the mechanism (`table_name()`), traits declared on the implementing side, and vendor types in a library's public API. Add a trait only for Noback's cases (trimmed public surface, I/O, isolated third-party code, an open set, a replaceable piece of a composed graph); otherwise a concrete struct. Flag `FooService` + `FooServiceImpl` pairs.  
7. **Packages (when crates or `Cargo.toml` change):** release/reuse (breaking public changes in a minor release, missing deprecation path, growing parameter lists, overly tight version requirements); common reuse (feature strata, bundled integrations, bonus features, implicit dependencies); common closure (one crate changing for dependency, layer, and business reasons); no cycles between modules or via dev-dependencies; dependencies point toward more stable crates; stable crates are abstract, and no concrete crate sits under most of the workspace.  
8. **Metrics when useful:** compute instability `I = C-out / (C-in + C-out)` and abstractness `A = C-abstract / (C-concrete + C-abstract)` over the workspace for crates in question, and report the distance `|A + I − 1|` as a hint, never a gate.  
9. **CUPID lens:** check composability (narrow but sufficient surface, intention-revealing, minimal dependencies), a clear outside-in purpose, predictability, idiom, and domain language and structure — only where another stage does not already own the point.  
10. **Findings:** skeptic concern format, naming the principle **and** the change pressure, path:line, what the next change would cost, and the smallest fix. LETTER options for real design forks (enum vs trait, split vs keep, invert vs depend directly). When run inside skeptic, merge or drop findings another stage already reported.  
11. When in scope: `solid: ok` (one line naming what was checked) or findings.  

## Do not

- Report an acronym without a change pressure ("violates SRP")  
- Demand traits for closed sets the crate owns, one trait per struct, or a trait whose only other implementation is a mock — the architecture defaults (enum, one `#[cfg(test)]` seam, passed in at the app edge) win  
- Ask to split units that change together, or into pieces that must be read together (→ complexity owns depth and conjoined splits)  
- Repeat component layout, public surface, data ownership, or "passed in, not built inside" (→ architecture), names (→ naming), Rust style (→ conventions), telemetry (→ observability)  
- Propose new crates for an application's internal layers or subdomains — modules do that job  
- Gate on metrics  

## Output section title

`## SOLID` — the extra section after stage 10 when skeptic runs (after `## Async` when both appear); `## SOLID` when run alone.
