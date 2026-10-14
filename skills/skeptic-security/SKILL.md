---
name: skeptic-security
description: >
  Security review: least privilege (identities, authentication, authorization,
  safe proxies, multi-party approval, emergency access, audit), trust boundaries
  (untrusted input, injection, safe-by-construction query types, parsing into
  validated types), credentials and cryptography, attacker-driven load, an
  understandable security model, and supply chain and deployment. Use when the
  user asks for a security review, or when skeptic stage 2 (architecture) runs
  and the diff touches a trust boundary.
---

# Skeptic security

**Question:** If a caller, a message, a credential, a dependency, or an insider turns hostile or makes a mistake — what can they read, change, or break, and would anyone know?

## When it runs

Run when the diff touches any of: input from outside the component (requests, messages, files, data read back from storage), authentication or authorization, credentials or keys, cryptography or TLS settings, identities and permissions (service accounts, roles, bucket policies), queries or commands built from data, admin or debug endpoints, dependencies or build and deployment configuration, or logging of user data. Otherwise report `security: none` with a one-line reason.

## Load

Load only what the diff touches:

1. `references/least-privilege.md` — small APIs, identities, authentication, authorization (action + arguments, deny by default, fail closed), safe proxies, multi-party approval, emergency access, audit  
2. `references/trust-boundaries.md` — assume nothing about callers, untrusted messages and stored data, injection and `TrustedSqlString`-style query types, parsing into validated types, safe deserialization  
3. `references/cryptography-and-data-handling.md` — keys and tokens out of source, redacting types, rotation, sensitive data in logs, cryptography, TLS and certificates  
4. `references/denial-of-service.md` — expensive operations, early rejection, amplification, caller-controlled fan-out, deadlines, fail-static policy  
5. `references/understandability.md` — invariants, small trusted base, centralized security, types that carry facts, `unsafe`  
6. `references/supply-chain-and-deploy.md` — dependencies, pinning, provenance, deploy gates, config as reviewed code, rolling out security changes  

## Do

1. **Boundaries:** list where untrusted data enters the change — requests, messages, files, and values read back from storage — and check each is validated inside the component, not "upstream".  
2. **Injection:** flag queries, commands, paths, and templates built from data by string formatting. Prefer APIs where unsafe mixing does not compile (`sqlx::query!`, a query builder that takes only `&'static str` fragments, `Command::arg`, validated path and key types); flag new uses of raw or unchecked escape hatches without review.  
3. **Parsing:** untrusted values are parsed once into validated types; deserialization has size and depth limits; cheap checks (size, schema, authorization) run before expensive work.  
4. **Authentication:** callers are identified by an established cryptographic mechanism; flag identity taken from unverified headers, parameters, or message fields, and access granted by network location.  
5. **Authorization:** every method authorizes the action **and** the resource ids it receives, through one framework with deny-by-default; backends acting for a user check that user's credential; security checks fail closed.  
6. **Privilege:** each job runs as its own identity with only the permissions its function needs — read access scoped like write access; destructive and admin operations on stricter permissions; no shared, ambient, or wildcard credentials.  
7. **Privileged paths:** admin and debug endpoints are small, specific, authenticated, authorized, and audited; proxies are the only path and act as the caller; unsafe actions need a second approver; emergency bypasses alert.  
8. **Credentials and data:** no credentials in source or config; credential and personal-data fields use redacting types; rotation works without a restart; no sensitive payloads, tokens, or user metadata in logs; TLS with verification everywhere.  
9. **Cryptography:** high-level library primitives only; authenticated encryption; no hashing of small value spaces as anonymization; stateless tokens signed, expiring, revocable, and bound.  
10. **Attacker cost:** expensive operations bounded per caller; no amplification or caller-sized fan-out; deadlines on every request and sub-request; protection policy fails static.  
11. **Supply chain and rollout:** new dependencies maintained and pinned; deploys verify the artifact; access rules and config change only through the pipeline; security changes small, flag-guarded, dry-run first, and rolled out gradually.  
12. **Findings:** skeptic concern format with the hazard named (injection, missing authorization, broken access scope, credential exposure, fail-open check, amplification, …), path:line, what an attacker or a mistake can do, and the smallest fix. LETTER options for real design forks (e.g. per-handler checks vs framework policy, proxy vs direct access).  
13. When in scope: `security: ok` (one line naming the boundaries checked) or findings.  

## Do not

- Credentials in errors or logs as pass/fail → hard rules (`HR-secret-in-error`); reference it  
- Generic overload protection (load shedding, quotas, backlogs) → `skeptic-reliability`; here only attacker-driven cost  
- Penetration-testing essays, threat models for code the diff does not touch, or compliance checklists  
- Demand controls out of proportion to the data and exposure (an internal read-only tool on public data) — say why the risk is acceptable instead  

## Output section title

`### Security` inside `## 2. Architecture` when skeptic runs; `## Security` when run alone.
