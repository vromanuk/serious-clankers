# Package principles, applied to crates and modules

Reference for reviewing how code is grouped into packages and how packages depend on each other. In Rust, a **package** is a crate (published, or a workspace member); its definition file is `Cargo.toml`. Modules inside a crate are Noback's "namespaces": structure without a separate release.

Martin: "The principles, however, focus very tightly on dependency management." A dependency on a package "is a dependency upon everything within the package."

---

## 0. When packages are the right tool

- **Packages are for reuse and independent release.** Inside one application, layers and subdomains are modules, not crates: packaging every layer × subdomain pair "will soon end up with an unmaintainable application that's very resistant to change".
- **Every package has a cost** — releases, versioning, issues, docs. "Releasing hundreds of tiny packages is something you can't possibly do."
- **The cohesion principles pull against each other** (§ 1.4). Early in a project, group by what changes together; split for outside reusers once the design settles.
- **Composition at the app edge** and component layout are the architecture stage's; this file covers grouping and dependency direction between packages.

---

## 1. Cohesion: what belongs in a package

### 1.1 Release/reuse equivalence

> "The granule of reuse is the granule of release."

Release only what can reasonably be reused, and you can only reuse what is actually released and maintained: "If the release process of a package is unmanageable or not managed at all, it can not be properly reused."

**Flag:**
- A new crate "for reuse" with one consumer and no owner for releases.
- A public item removed or changed in a minor or patch release. Removals wait for a major version; in the meantime keep a forwarding item marked `#[deprecated(since = "…", note = "use X instead")]`, or `pub use new::Name as OldName;` after a move. `cargo-semver-checks` catches many of these.
- A new parameter on a public function. Rust has no default arguments, so this always breaks callers — use an options struct with `Default` (and `#[non_exhaustive]`), or a builder, from the start.
- Public structs with all-`pub` fields built by literal: adding a field breaks every user. Prefer private fields with a constructor.
- A public function with a hidden side effect (creating directories, opening connections) beyond its documented job: users come to rely on it, and removing it later breaks them.
- Library dependency requirements tighter than needed (`foo = "=2.4.3"`, or a raised minimum with no code reason).
- A released crate with no README, license, or changelog entry for the version, or no tests in CI.

### 1.2 Common reuse

> "Classes that are used together are packaged together." — and, more importantly, do not package together what is not used together.

**Flag:**
- **Feature strata**: top-level modules of one crate that never use each other, only a shared core — really several libraries in one.
- **Mutually exclusive features** in one crate (cargo features that `compile_error!` when combined).
- **Integrations bundled with the core**: `optional = true` dependencies behind cargo features for backends that evolve on their own schedule. Move each into its own crate that depends on the core.
- **Bonus features**: a generic helper (cache wrapper, retry layer) living inside a domain-specific crate — someone using only the helper pulls in everything else.
- **Implicit dependencies**: using a crate through another crate's re-export (`b::c::Thing`) instead of declaring `c` — the day `b` drops `c`, this breaks.
- **Unbuilt code**: `cfg(feature = …)` paths never compiled in CI.

Noback's two questions when adding a type to a package: "Does the class introduce a dependency?" (if it is optional, the type belongs in a new package that declares it) and "Would the class be useful without the rest of the package?" (if so, it belongs in its own package).

### 1.3 Common closure

> "The classes in a package should be closed together against the same kinds of changes. A change that affects a package affects all the classes in that package."

The kinds of change to close against: a change in a dependency, a change in an architectural layer, and a change in the business.

**Flag:**
- A dependency upgrade or replacement that edits two files in a crate of fifty: those two belong in their own crate or module.
- One reusable package containing domain model, persistence, transport, and presentation.
- A domain-types crate that depends on `axum`, `sqlx`, or `tonic`, so users of the model must take the framework too.
- Core business logic extracted into a "reusable" crate. Reuse works for generic subdomains; "a core domain needs a custom modeling effort to get it just right."
- A single requirement change that needs coordinated releases of many crates.

### 1.4 The tension between them

The three cohesion principles "cannot simultaneously be satisfied": reuse and common reuse make life easier for users, common closure for maintainers. A package pushed to one corner neglects the others.

- **Reuse broken:** hard to add as a dependency and to keep up to date.
- **Common reuse broken:** you pull in a lot of code to use one or two types.
- **Common closure broken:** the package holds responsibilities from many unrelated fields.

---

## 2. Coupling: how packages depend on each other

### 2.1 Acyclic dependencies

> "The dependency structure between packages must be a directed acyclic graph; that is, there must be no cycles in the dependency structure."

Cargo rejects cycles between normal crate dependencies, so look for the forms it allows:

- cycles between **modules** inside a crate (`a` uses `b` uses `a`);
- cycles through `dev-dependencies`;
- runtime cycles between services or crates through callbacks or shared configuration.

Hidden coupling counts too: types used only inside function bodies, global registries, and free functions from undeclared crates.

**Breaking a cycle, in order:**

1. **Pseudo-cycle?** If the two directions involve unrelated types, move types instead of redesigning.
2. **Invert one edge** with a small trait placed on the side that uses it, or in its own small crate.
3. **Bridge crate** (`foo-bar`) for two packages that are useful together but should not know each other.
4. Avoid hiding *central* behavior behind a generic event bus just to remove an edge: `dispatch()` says nothing about what happens.

Do not break an object-level cycle with an `Option<Arc<Other>>` filled in after construction: for a while, one object is invalid. Inject only the part that is needed, or make the relation one-directional.

### 2.2 Stable dependencies

> "The dependencies between packages in a design should be in the direction of the stability of the packages. A package should only depend upon packages that are more stable than it is."

"Stable" here means hard to change because others depend on it — not semver stability, and not code quality.

- **Instability:** `I = C-out / (C-in + C-out)` — C-out counts external types this package depends on, C-in counts external types that depend on this package. 0 is maximally stable, 1 maximally unstable. (Martin's form: `I = Ce / (Ca + Ce)`.) Compute it only over the crates in the workspace or application.
- Lay out the graph with stable crates at the bottom: every arrow should point down.

**Flag:**
- A widely used core crate gaining a dependency on a fast-changing feature or integration crate.
- Framework, ORM, or SDK dependencies in a core crate. Define the trait in the core and put the adapter in a leaf crate — leaf adapters with `I = 1` are fine, since nothing depends on them.
- Repeated "fix build after bumping X" changes: first question the choice of dependency.

### 2.3 Stable abstractions

> "Packages that are maximally stable should be maximally abstract. Instable packages should be concrete. The abstraction of a package should be in proportion to its stability."

- **Abstractness:** `A = C-abstract / (C-concrete + C-abstract)` — in Rust, roughly public traits versus concrete structs, enums, and functions (a rough proxy: generics make abstractness a matter of degree).
- **Main sequence:** `I + A` should be near 1. Martin measures the distance as `D′ = |A + I − 1|` (0 on the line, 1 farthest away).
- **Corners to flag:**
  - **Concrete and stable** (A ≈ 0, I ≈ 0) — a concrete crate much of the workspace depends on: every change ripples. Martin's "zone of pain". Fix by dependency inversion in the crates that depend on it.
  - **Abstract and unstable** (A ≈ 1, I ≈ 1) — traits nobody implements or uses: likely dead code. Remove it.
- **Interfaces belong in stable packages**: "The interface inherits the instability of its containing package." A plugin trait defined inside an app or integration crate is not safe to depend on.
- **No single `traits` or `contracts` crate** for unrelated subsystems — that breaks common reuse. Prefer small ones per feature, and keep internal-only traits `pub(crate)`.
- Application crates are allowed to be concrete and unstable: nothing depends on them.

Treat the numbers as hints, not gates: "reliance upon them as the sole indicator of a sturdy architecture would be foolhardy."

---

## Sources

- Principles of Package Design (Matthias Noback, Apress, 2018) — https://doi.org/10.1007/978-1-4842-4119-6
- Design Principles and Design Patterns (Robert C. Martin, 2000) — https://web.archive.org/web/20150906155800/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf
- The Principles of OOD (Robert C. Martin) — http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod
- https://github.com/obi1kenobi/cargo-semver-checks — cargo-semver-checks
