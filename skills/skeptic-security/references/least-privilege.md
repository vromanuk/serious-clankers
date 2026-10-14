# Least privilege: identity, authentication, authorization, audit

Reference for reviewing who can do what, with which credential, and how anyone would know afterwards. Assume every credential may be stolen and every person may make a mistake: any privilege given to an insider is available to an attacker.

---

## 1. Small, specific interfaces

### 1.1 Small functional APIs, not open-ended ones

Expose specific operations ("restart job", "push config with hash X", CRUD on an id) instead of interfaces that accept arbitrary commands, scripts, or query text. A small API can be granted to exactly the function that needs it and audited exactly; a shell or free-form query language cannot.

- **Flag:** an admin or debug endpoint taking a command string, script, or SQL; automation that SSHes into hosts for one well-defined task; one RPC that switches on a free-form `action` field.

### 1.2 Destructive operations get a stricter permission than reads

Delete, purge, bulk, reprocess, setup/teardown, and emergency operations get narrower permissions than ordinary reads, and should not live on the public listener with user-traffic auth.

### 1.3 Read access is as dangerous as write access

Overly broad *read* permission is how mass data breaches happen. Scope reads to the prefix, topic, or table the job needs. "It's read-only" is not a reason for wide access.

---

## 2. Identities

### 2.1 Different jobs, different identities

Each service, per environment, runs as its own meaningful identity (`service-consumer-prod`), not a shared or anonymous one. Anyone who compromises one job can act as its identity anywhere; a shared identity turns one compromise into all of them.

- **Flag:** a consumer, an admin tool, and a batch job sharing one service account; one identity across production and non-production; identity inferred from IP or port.

### 2.2 Automation credentials do one task

Deploy, config push, and migration automation get a credential for that task only — not root, cluster-admin, or a wildcard role. A leaked automation credential otherwise equals a compromise of every target.

### 2.3 No ambient authority

No standing root or admin power for people or workloads. Containers do not run as root or privileged; service accounts do not hold owner roles "to avoid permission errors".

### 2.4 Identifiers in access rules are never reused

Access rules keyed on identifiers that can be reassigned (an email address given to a new hire, a hostname, an IP) hand old privileges to the new holder.

### 2.5 Separate data access

Each service accesses only its own data. When several services share one database or storage credential, a bug in any of them exposes everything.

---

## 3. Authentication and authorization

### 3.1 Network location grants nothing

Authenticate and authorize every request on credentials, never on being inside the internal network. "Internal only" endpoints, VPC membership, and IP allowlists are not access control.

### 3.2 Authenticate with an established cryptographic mechanism

mTLS, OAuth, signed tokens — not identifiers the caller asserts about itself.

- **Flag:** caller identity read from a query parameter, an `X-User` / `X-Service` header, or a message field that no signature or transport verifies.

### 3.3 Authorize the action and its arguments

"Authenticated" is not "authorized". Decide whether *this* identity may perform *this* method on *this* resource. Resource ids taken from a request or message (tenant, device, object key) are checked against the caller.

```rust
// Worse: any authenticated caller can read any tenant's data
async fn get_signals(&self, request: Request<GetSignals>) -> Result<Response<Signals>, Status> {
    let caller = authenticated_identity(&request)?;
    let tenant = request.get_ref().tenant_id.clone();
    Ok(Response::new(self.store.read(&tenant).await?))
}

// Better: the caller's access to that tenant is checked first
    let tenant = TenantId::parse(&request.get_ref().tenant_id)?;
    self.policy.require(&caller, Action::ReadSignals, &tenant)?;  // deny unless allowed
```

### 3.4 Authorization in a framework, deny by default

Enforce authorization in one interceptor or middleware with a declared policy for every method, denying anything not explicitly allowed. Ad-hoc `if user == "admin"` checks get forgotten, and a service with authentication but no authorization lets every authenticated client do everything. A reviewer can then understand the controls without reading every handler.

- **Flag:** a new service with mTLS or token validation but no authorization policy; a new method with no policy entry.

### 3.5 Backends need the end user's context

For requests made on a user's behalf, the backend verifies a credential tied to that user (or a scoped ticket derived from it) instead of returning data for any user id the calling service supplies — otherwise compromising the frontend exposes every user. At each hop, decide whether the workload or the end user is the authority.

### 3.6 Security checks fail closed

When an authorization, signature, or revocation check cannot complete, deny. Failing open lets an attacker switch security off by overloading the check. A control may degrade only to a cheaper one that is at least as strong.

- **Flag:** an error branch around a permission check that returns allow; a missing or failed-to-load access list treated as "allow all"; an authorization timeout that falls through to success.

---

## 4. Safe proxies, multi-party approval, emergency access

### 4.1 The proxy must be the only path

A control proxy for privileged operations (audit, access rules, approvals, rate limits) helps only if the target rejects direct connections except in an emergency. The proxy forwards and acts under the *caller's* identity, not a powerful identity of its own.

- **Flag:** a gateway that calls backends with its own broadly scoped token for every user request; backend logs that show only the proxy's identity.

### 4.2 Two people for unsafe actions

Require a second person's approval for bulk deletes, user-data exports, access-rule changes, and production data fixes, and show the approver exactly what will run. The requester cannot approve their own request.

### 4.3 Emergency access ("breakglass") is rare and loud

Any bypass of normal authorization is limited to a small group, alerts on every use, is tested, and is reviewed afterwards. Routine use means a safer normal API is missing. Emergency access must also work when the normal access stack (single sign-on) is down.

- **Flag:** a `SKIP_AUTH` / `--force` flag or environment variable with no alert; runbooks that use direct database or host access for routine tasks.

### 4.4 Sensitive records blocked by default

Deny access to individual customer records unless a verified business need covers that record, ideally time-bound. Tie access to a structured reference (ticket, incident id) that the system checks against the record — free-text reasons cannot be verified.

---

## 5. Audit

### 5.1 Log every privileged action, granularly

For each data access or privileged action, record actor, target, exact operation (for example "pushed config with hash …"), time, and justification. "Opened a session" says nothing; shell history is easily bypassed. Applications log security-relevant actions: data writes, ownership or state changes, account activity, access-rule changes.

### 5.2 Ship logs off the host, keep them long enough

Attackers delete local logs. Send audit logs to a central store the service cannot modify or delete, set retention explicitly (intrusions are often found months later), and when storage runs short, degrade (summarize) rather than stop. Decide what happens when audit logging fails, and alert on it.

### 5.3 Lock down debug endpoints

Debug endpoints (impersonation, raw data access, dumps, profiling) need authentication, authorization, and auditing like any other privileged interface.

---

## 6. Test the boundaries

Test that each role can do its job **and no more**: negative cases (bulk read, delete, cross-tenant access, staging credentials against production) must fail. Separation silently regresses without such tests.

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- Understanding Distributed Systems, 2nd edition (Roberto Vitillo) — https://understandingdistributed.systems/
