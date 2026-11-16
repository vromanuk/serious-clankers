# Lint baseline

A rule a machine can check belongs in the linter, not in every review. This file says what lint setup a Rust
crate should have, how silenced lints are reviewed, and which clippy lints catch findings a reviewer would
otherwise make by hand.

**Sources:** Apollo GraphQL, *Rust Best Practices* handbook, chapter 2 (clippy and linting), and its
`rust-best-practices` skill; [Cargo `[lints]` table](https://doc.rust-lang.org/cargo/reference/manifest.html#the-lints-section);
[clippy lint list](https://rust-lang.github.io/rust-clippy/master/index.html).

## 1. The crate has a lint baseline

Check that the workspace (or the crate) sets its lints in `Cargo.toml`, and that each member crate opts in. Flag a
new crate with no `[lints]` table, and a crate in a workspace without `[lints] workspace = true`.

```toml
# Workspace Cargo.toml
[workspace.lints.rust]
future_incompatible = { level = "warn", priority = -1 }
nonstandard_style = { level = "deny", priority = -1 }

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }
allow_attributes_without_reason = "deny"
undocumented_unsafe_blocks = "deny"
redundant_clone = "warn"

# Each member crate's Cargo.toml
[lints]
workspace = true
```

**Priority:** Cargo applies lints from the lowest `priority` to the highest, so a higher number overrides a lower
one. Groups (`all`, `pedantic`) get `-1`, so single lints set at the default `0` override them. Apollo's handbook
gives the groups high numbers and describes the order the other way round; do not copy that part.

A library crate also sets `#![warn(missing_docs)]` (or `missing_docs = "warn"` under `[lints.rust]`), so every
public item has a doc comment (`skeptic-comments`).

Run in CI: `cargo clippy --all-targets --all-features --locked -- -D warnings`. `--all-targets` includes tests,
benches, and examples; `--locked` fails when `Cargo.lock` is out of date.

## 2. Silenced lints say why, and expire

- Prefer fixing the code over silencing the lint.
- When a lint must be silenced, silence it **locally**, with `#[expect(...)]` and a reason:

```rust
// Worse: stays forever, even after the code no longer triggers the lint; nobody knows why
#[allow(clippy::large_enum_variant)]
enum Message { Ping, Frame([u8; 4096]) }

// Better: warns when the lint stops firing, so the attribute is removed with the reason it existed for
#[expect(clippy::large_enum_variant, reason = "Frame is the hot path; boxing adds an allocation per message")]
enum Message { Ping, Frame([u8; 4096]) }
```

- Flag a crate-wide `#![allow(clippy::...)]` added by the diff. It hides every future hit too.
- An `#[allow]` or `#[expect]` with no `reason` is `HR-silenced-lint-without-reason` (`skeptic-hard-rules`).

## 3. Name the lint instead of checking by hand

When a finding is one a clippy lint catches, report it once, name the lint, and suggest turning the lint on if the
crate does not have it. The next review then does not need to look for it. Some of these lints are in the
`pedantic`, `nursery`, or `restriction` groups, so they are off by default; enable them by name.

| Finding a reviewer would make | Lint |
|---|---|
| `match` or `if let` that only returns early on the other case | `manual_let_else` |
| `match` on `Result`/`Option` that only propagates the error | `question_mark` |
| `match` turning `Option` into `Result` | `manual_ok_or` |
| `unwrap_or(Vec::new())`, `ok_or(format!(...))`: the default is built even when not needed | `unwrap_or_default`, `or_fun_call` |
| `.collect()` into a `Vec` only to iterate it again | `needless_collect` |
| `.clone()` whose original is never used again | `redundant_clone` |
| `.clone()` on a `Copy` type | `clone_on_copy` |
| `.map(\|x\| x.clone())` | `map_clone` |
| `.fold(0, \|a, b\| a + b)` instead of `.sum()` | `unnecessary_fold` |
| `&Vec<T>` / `&String` parameter | `ptr_arg` |
| owned parameter only read inside the function | `needless_pass_by_value` |
| small `Copy` value passed by reference | `trivially_copy_pass_by_ref` |
| large value passed by value | `large_types_passed_by_value` |
| one enum variant much larger than the others | `large_enum_variant` |
| a type that is both `Copy` and `Iterator` | `copy_iterator` |
| several `bool` parameters | `fn_params_excessive_bools` |
| public fallible function with no `# Errors` section | `missing_errors_doc` |
| public function that can panic with no `# Panics` section | `missing_panics_doc` |
| public `unsafe fn` with no `# Safety` section | `missing_safety_doc` |
| `unsafe` block with no `// SAFETY:` comment | `undocumented_unsafe_blocks` |
| `#[allow]` / `#[expect]` with no reason | `allow_attributes_without_reason` |

Check a lint's meaning with `cargo clippy --explain <lint>` before citing it.

**Not lints, still judgment:** whether an `unwrap` is a bug that should stop the program or an operating error that
should be returned (`skeptic-complexity` § 4). `unwrap_used` / `expect_used` flag every call, including the
`expect("why this cannot happen")` the complexity stage asks for, so suggest them only for crates that already
ban panics.
