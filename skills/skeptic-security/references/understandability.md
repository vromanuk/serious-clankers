# Understandability: a security model a reviewer can check

Reference for reviewing whether the security properties of a change can be understood — and so trusted — without tracing the whole system.

---

## 1. State the invariants

Write down what must always be true ("only callers authorized for tenant T read T's data") and argue it holds for **all** inputs, including malformed and malicious ones. Tests cover typical inputs; a security property must survive crafted ones.

- **Flag:** a security-relevant change justified only by happy-path tests, with no statement of what must stay true.

## 2. Keep the trusted part small

The trusted computing base is every component that can break the security policy. Keep out anything not needed to uphold it: unrelated features do not run in the same process or under the same credential as code handling sensitive data.

## 3. Put cross-cutting security in one place

Authentication, authorization, and audit logging live once, in the RPC framework or middleware, configured per service — not copied into each handler. Developers cannot forget a check, and a reviewer understands the controls from one place (`least-privilege.md` § 3.4).

## 4. Make properties visible in the code

- **Types carry facts.** Validated inputs, tenant ids, object keys, and secrets get their own types (`trust-boundaries.md` § 3, `cryptography-and-data-handling.md` § 1.2). Confusable primitives (several `String` or `u64` parameters, bare durations) become distinct types so mix-ups fail to compile.
- **Handle failures where they are checked.** Return early on authorization and validation failures instead of handling them several levels of nesting away, where error branches go untested and get swapped.
- **Structure code for review.** If deciding whether a value is trusted means reading every transitive caller, restructure.
- **Remove speculative code.** Flags, modes, and parameters with no caller must still be understood, tested, and secured; simpler code has fewer security bugs.

## 5. Memory safety

Memory-safety bugs are the majority of security vulnerabilities, and memory-safe languages remove the class by default. In Rust that guarantee ends at `unsafe`:

- Every `unsafe` block states the invariant that makes it sound (a `// SAFETY:` comment; clippy `undocumented_unsafe_blocks` enforces it).
- New `unsafe`, FFI bindings, or C/C++ dependencies are high-risk and get explicit review.
- Run Miri or sanitizers in CI for crates that contain `unsafe`.

## 6. Identities and ordering a reviewer can follow

Workloads run as meaningful, specific identities (`least-privilege.md` § 2), so access rules are readable and audit is possible. Mutating operations carry request ids so duplicates are recognized and the API keeps a simple mental model (`skeptic-reliability` → `downstream.md` § 3).

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://rust-lang.github.io/rust-clippy/master/index.html#undocumented_unsafe_blocks — clippy `undocumented_unsafe_blocks`
- https://github.com/rust-lang/miri — Miri
