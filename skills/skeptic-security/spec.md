# Skeptic Security

## Intent

Review whether a change limits what a hostile or mistaken caller, message, credential, dependency, or insider can do: least privilege and authorization, trust boundaries and injection, credentials and cryptography, attacker-driven load, an understandable security model, and the supply chain. Runs standalone, or inside skeptic stage 2 (architecture) as a `### Security` subsection when the diff touches a trust boundary.

## Triggers

- **SHOULD** apply when the user asks for a security review, or when skeptic stage 2 runs and the diff touches untrusted input, authentication or authorization, credentials or keys, cryptography or TLS, identities and permissions, queries or commands built from data, admin or debug endpoints, dependencies or deployment configuration, or logging of user data.
- **SHOULD NOT** apply to diffs with none of these (report `security: none` with a reason).

## Behaviors

### Behavior: Injection-safe construction

The agent SHALL flag queries, commands, paths, and templates built from data by string formatting, and SHALL prefer APIs where mixing input and code does not compile.

#### Scenario: SQL built with format!

- **GIVEN** a handler that builds an UPDATE statement with `format!` from request fields and runs it with `sqlx::query(&sql)`
- **WHEN** reviewing security
- **THEN** the agent flags injection, and suggests a literal query with bound parameters, or `sqlx::query!`, so a runtime string cannot be passed as SQL

#### Scenario: New raw-query escape hatch

- **GIVEN** a database wrapper gains a public `fn query(sql: String)` next to its literal-only builder
- **WHEN** reviewing security
- **THEN** the agent flags the unchecked path and suggests one clearly named, reviewed escape-hatch type with new uses blocked by a lint

### Behavior: Authorization of action and arguments

The agent SHALL flag handlers that authenticate the caller but act on a resource id from the request or message without checking the caller's access to that resource, and authorization that fails open.

#### Scenario: Any authenticated caller reads any tenant

- **GIVEN** a gRPC method that verifies the caller's mTLS identity and then returns data for the tenant id in the request
- **WHEN** reviewing security
- **THEN** the agent flags missing authorization of the resource and suggests a deny-by-default policy check on caller, action, and tenant

#### Scenario: Permission check error allows the request

- **GIVEN** a permission check whose error branch logs and continues
- **WHEN** reviewing security
- **THEN** the agent flags the fail-open check and asks for a deny on error

### Behavior: Least privilege

The agent SHALL flag identities shared across jobs or environments, permissions broader than the job's function (including broad read access), and admin or destructive operations on the same permission as ordinary reads.

#### Scenario: Bucket-wide read for one prefix

- **GIVEN** a new service account granted read on a whole bucket when the service reads one prefix
- **WHEN** reviewing security
- **THEN** the agent flags the scope and asks for read access limited to that prefix

### Behavior: Credentials and sensitive data

The agent SHALL flag credentials in source or configuration, credential or personal-data fields that derive `Debug` or `Serialize` as plain strings, disabled certificate verification, and sensitive payloads in logs.

#### Scenario: Token in a Debug-derived config

- **GIVEN** a config struct with `token: String` that derives `Debug` and is logged at startup
- **WHEN** reviewing security
- **THEN** the agent flags credential exposure and suggests a redacting type such as `secrecy::SecretString`, read explicitly where the token is sent

### Behavior: Attacker-driven cost

The agent SHALL flag expensive operations with no per-caller bound, expensive work before cheap checks, and caller-controlled fan-out.

#### Scenario: Decompression before the size check

- **GIVEN** a handler that decompresses a request body and then checks its size
- **WHEN** reviewing security
- **THEN** the agent flags that an attacker can make the server do the expensive work, and suggests a limit on compressed and decompressed size before decoding

### Behavior: Proportionate findings

The agent SHALL NOT demand controls out of proportion to the data and exposure, and SHALL say why the risk is acceptable instead.

#### Scenario: Internal tool on public data

- **GIVEN** an internal command-line tool that formats public documentation files
- **WHEN** reviewing security
- **THEN** the agent reports `security: none` or one line, without demanding authorization frameworks or audit logs

## Constraints

### Constraint: Stage boundary

The agent MUST NOT repeat the `HR-secret-in-error` hard rule, generic overload protection (skeptic-reliability), or alert design (observability); it cross-references them instead.

<!-- skillet-version: 1.7.0 -->
