# Keys, tokens, sensitive data, and cryptography

Reference for reviewing how a change handles keys, tokens, personal data, and encrypted connections. `HR-secret-in-error` is the pass/fail floor for keys and tokens in errors and logs.

---

## 1. Keys and tokens

### 1.1 Never in source or version control

Keys, passwords, and tokens do not appear in repository files, Helm values, Dockerfiles, CI configs, or test fixtures that point at real systems. Load them through a secret manager or key management service, readable only by the services that need them.

### 1.2 Distinct types for keys, tokens, and sensitive data

Wrap tokens, keys, and personal data in dedicated types that refuse to print or serialize, instead of plain `String` fields on structs that derive `Debug` or `Serialize` and end up in logs, telemetry, or storage.

```rust
// Worse: the token is printed by any {:?} of the config, and serialized with it
#[derive(Debug, Serialize)]
struct UploaderConfig { endpoint: Url, token: String }

// Better: Debug prints "[REDACTED]", and reading the value is explicit
#[derive(Debug)]
struct UploaderConfig { endpoint: Url, token: secrecy::SecretString }
// use: token.expose_secret() — greppable, and only where the token is sent
```

### 1.3 Rotation is a normal operation

Design so keys and tokens can be rotated without a restart: reload them, accept an overlap window (a key id or version), and confirm the old one stops working. A key loaded once at startup, with a single hardcoded value, makes rotation a crisis. Tokens with wall-clock expiry need monitored rotation and an alert well before expiry (`skeptic-reliability` → `recovery.md` § 3).

### 1.4 Never over cleartext

Send passwords and bearer tokens only over authenticated, encrypted channels. Do not put tokens in URLs (they end up in access logs and proxies).

---

## 2. Sensitive data in logs and telemetry

- **A policy for what may be logged.** Do not log full request or message payloads, tokens, keys, or personal data by default; redact known sensitive fields.
- **Metadata is sensitive too.** Who accessed what, when, and how often reveals a lot even with payloads removed. Access logs with user and resource ids need the same access control as the data.
- **Location and personal telemetry is high-sensitivity.** Classify it, and do not give it the access controls of operational data.

---

## 3. Cryptography

### 3.1 Don't build your own

Use a well-established library with high-level primitives (an AEAD API, a TLS library, a token library) instead of composing primitives, inventing token or envelope formats, or choosing modes and padding by hand.

### 3.2 Authenticated encryption; nonces managed by the library

Encryption without integrity protection can turn a service into a decryption oracle; nonce reuse breaks AES-GCM. Use an authenticated-encryption API that manages nonces, and do not reveal in errors whether decryption or padding failed.

### 3.3 Hashing does not hide small value spaces

Hashing a phone number, VIN, card number, or account id does not anonymize it: the set of possible values is small enough to enumerate. Use keyed or real encryption, or a different design.

### 3.4 A library is not a threat model

"We use library X" prevents low-level mistakes, not the wrong primitive (signing where encryption is needed) or no cryptography at all. Check that the primitive matches the threat.

### 3.5 Stateless tokens carry what keeps them safe

A token issued instead of server-side state needs a signature (cannot be forged), an issue time or expiry (not usable forever), an identifier (revocable), and a binding to its holder (not shareable).

---

## 4. Transport

- **TLS everywhere**, including internal traffic that never crosses the internet — the CPU cost is negligible.
- **Verify certificates.** Never disable verification (`danger_accept_invalid_certs`, custom verifiers that always pass). Without it, encryption does not stop impersonation.
- **Certificates expire.** An expired certificate is a single point of failure: automate renewal and alert before expiry.
- **Reuse connections.** A new TLS connection costs extra round trips; build clients once and share them.
- **Integrity in transit.** TCP's checksum misses some corruption on large transfers; TLS integrity, or an end-to-end checksum, catches it.

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
- https://docs.rs/secrecy/latest/secrecy/ — `secrecy` (types that keep tokens and keys out of `Debug` output)
