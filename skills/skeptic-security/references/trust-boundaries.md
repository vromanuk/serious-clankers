# Trust boundaries: untrusted input, injection, parsing, safe-by-construction APIs

Reference for reviewing where data crosses from something you do not control into something that acts on it.

---

## 1. Where the boundaries are

### 1.1 Assume nothing about callers

A component should make no assumptions about its callers. If correctness depends on every caller honoring a precondition, trusting the property means reviewing every call site in the system. Validate inside the component — even when the only callers are "our own services".

- **Flag:** an internal RPC or consumer that skips validation "because only our service calls it"; a doc comment "caller must ensure X" with no check.

### 1.2 Everything that crosses a boundary is suspect — including order

Network requests, queue and log messages, files, and IPC can be malformed, duplicated, reordered, or forged. The trusted part must uphold its rules even if everything outside misbehaves, maliciously or not.

- **Flag:** a consumer that assumes messages are well formed, in order, unique, or from a trusted producer; a state machine that accepts out-of-order events.

### 1.3 Data read back from storage is still untrusted

Stored injection: a value enters at the edge, passes through services, is saved, and is later read back and used in a query, template, path, or command. "It came from our database / topic / bucket" is not validation.

### 1.4 Each layer checks for itself

Design each layer to be independently resilient. An inner service that skips authorization or validation because "the gateway already checks it" leaves one check between untrusted input and a destructive write.

---

## 2. Injection: make the unsafe pattern impossible

### 2.1 Queries are developer-controlled code plus bound parameters

A query is safe from injection when it consists only of developer-written text, with external input supplied as bound parameters. String concatenation lets input become commands.

```rust
// Worse: request data becomes SQL
let sql = format!("UPDATE users SET pw_hash = '{hash}' WHERE reset_token = '{token}'");
sqlx::query(&sql).execute(&pool).await?;

// Better: the SQL is a literal, the values are bound
sqlx::query("UPDATE users SET pw_hash = $1 WHERE reset_token = $2")
    .bind(hash)
    .bind(token)
    .execute(&pool)
    .await?;
```

### 2.2 A guideline does not scale; a type does

"Always use prepared statements" needs every developer and reviewer to remember it forever. Design the API so mixing input and SQL **does not compile**. Google's version is `TrustedSqlString`: a query type that can only be built from string literals in the source code; values go in only as parameters.

In Rust:

- `sqlx::query!` / `query_as!` take the SQL as a literal and check it at compile time — a runtime string cannot be passed.
- A home-grown query builder can accept SQL fragments only as `&'static str`, with values through typed `bind` methods:

```rust
/// SQL built only from literals in this crate's source; values are bound, never spliced.
pub struct TrustedQuery { sql: String, params: Vec<Param> }

impl TrustedQuery {
    pub fn new(literal: &'static str) -> Self { /* … */ }
    pub fn append(mut self, literal: &'static str) -> Self { /* … */ }
    pub fn bind(mut self, value: impl Into<Param>) -> Self { /* … */ }
}
```

`&'static str` approximates "a literal": `String::leak` or `Box::leak` can still produce one from runtime data. Ban those in production code with clippy's `disallowed_methods`, so the bypass is visible.

- The same idea applies to every injection-prone sink: HTML (Google's `SafeHtml`), URLs (`SafeUrl`), shell commands (`Command::new(program).arg(value)`, never `sh -c` with a formatted string), file paths (a validated path type that cannot escape its base directory), and query or filter languages.

### 2.3 One named escape hatch, reviewed

Some features really need dynamic queries (an analytics tool that runs the owner's SQL). Route every bypass through one clearly named type or module — Google's `unsafequery.String` — so new uses stand out and get security review. Restrict the old unsafe API to existing callers and block new ones mechanically (a lint, visibility, or an allowlist).

- **Flag:** a wrapper exposing `fn query(sql: String)`; a new call to a raw or unchecked API without sign-off; scattered ad-hoc conversions from `String` to a "safe" type.

### 2.4 Ban risky APIs with compiler or lint errors

Compiler and lint errors give feedback while the author still has context; review-time findings come late and depend on the reviewer noticing. Use `#![forbid(...)]`, clippy `disallowed_methods` / `disallowed_types`, or a CI check.

---

## 3. Parse, don't pass strings around

### 3.1 Carry validation in types

A `String` asserts nothing about being a well-formed URL, object key, topic name, or tenant id. Parse once, at the boundary, into a type whose constructor enforces validity, and pass that type downstream. Then a reviewer checks the constructor and the final use, not every caller in between.

```rust
/// An object-storage key under this service's prefix, with no `..` segments.
pub struct ObjectKey(String);

impl ObjectKey {
    pub fn parse(raw: &str) -> Result<Self, InvalidKey> { /* validate once */ }
}

// The storage client takes ObjectKey, not &str.
```

### 3.2 Typed interfaces over free-form payloads

Define APIs with explicit methods and message types (protobuf, OpenAPI) instead of free-form JSON interpreted in business logic. Free-form payloads hide the real surface, let clients and servers drift apart, and block automated security checks.

- **Flag:** handlers taking `serde_json::Value` or a map of anything; a new API with no schema.

### 3.3 Deserialize untrusted data with formats built for it

Use formats and libraries designed for untrusted input (protobuf, Avro with a known schema, `serde` into concrete types), set size and depth limits, and disable risky parser features (external entities in XML, type-tagged polymorphic deserialization).

- **Flag:** no maximum body, message, or decompressed size; recursion depth unbounded; allocation sized by a caller-supplied length field.

### 3.4 Cheap checks before expensive work

Check size limits, schema, and authorization before allocating memory, decompressing, fully deserializing, or starting reads and writes. Work done for a request that later fails is lost to everyone else, and attackers can aim at it (`denial-of-service.md`).

---

## 4. Errors and debug features

Do not return internal error details (stack traces, SQL errors, internal hostnames) to clients. Gate all debug features behind one flag that deployment checks keep off for public users, instead of scattered toggles.

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://docs.rs/sqlx/latest/sqlx/macro.query.html — `sqlx::query!` (compile-time checked queries)
- https://rust-lang.github.io/rust-clippy/master/index.html#disallowed_methods — clippy `disallowed_methods`
