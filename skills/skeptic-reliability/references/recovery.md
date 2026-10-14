# Recovery: known good state, time, credentials, backups, failover

Reference for reviewing whether the system can get back to a good state after something breaks — including when the breakage is in the recovery tools themselves.

---

## 1. Known good state

### 1.1 A way back for every change

The first step of most recoveries is rolling back to the last known good state, quickly and safely. Code, config, and schema changes each need that path, or must be marked irreversible (`change-safety.md` § 1).

### 1.2 A minimum version for rollbacks

Unlimited rollback can reinstall a release with a known vulnerability; no rollback removes the way back to stability. Keep an ordered minimum acceptable version — a simple integer, not a hash — that rollback tooling enforces, and raise it only after the fixed release has proven stable.

### 1.3 Build for a second instance from day one

Recovery may mean standing up a new copy of a store, bucket, topic, or cluster. Hardcoded names and code that assumes it is the only writer to a global resource block that.

- **Flag:** hardcoded bucket, topic, table, or cluster names; manual turn-up steps not captured in code.

---

## 2. Time

### 2.1 Durations from a monotonic clock

Timeouts, leases, and elapsed time use a monotonic clock (`Instant`). Wall clocks (`SystemTime`) jump when synchronized and can run backwards.

### 2.2 Avoid wall-clock time for security and expiry decisions

A wrong clock — by accident or attack — can make revoked credentials valid again or reject good ones and cause an outage. Prefer rates, epochs or version numbers, and validity lists where they work.

### 2.3 No time bombs

Literal dates and fixed offsets in logic, and far-future expiries ("valid until 2038"), are failures scheduled for later — often during a recovery.

### 2.4 Replays check against the record's own time

Backfills and replays must evaluate expiry, TTLs, and signatures against when each record was created, not "now". Otherwise reprocessing old data fails or behaves differently from the live path.

```rust
// Worse: a replay of last week's records rejects them all as expired
if record.expires_at < SystemTime::now() { return Err(Expired); }

// Better: the check uses the time the record was produced
if record.expires_at < record.produced_at { return Err(Expired); }
```

---

## 3. Credentials and revocation

### 3.1 Rotate routinely, and prove the old key stops working

Rotate keys and credentials even when not required, so rotation works when an emergency demands it. Verifiers accept an overlap window (a key id), secrets are reloadable without a restart, and after rotation the old key is rejected.

### 3.2 Expiry must survive a stalled rotation

If rotation breaks before old credentials expire, expiry becomes an outage. Monitor rotation, alert well before expiry, and make sure the refresh path does not depend on the service whose outage it must survive.

- **Flag:** short-lived tokens refreshed only at process start; no alert on approaching expiry.

### 3.3 Revocation must not fail open, or take everything down

A revocation or validity check that times out to "valid" undoes the protection; a synchronous call to a central validity service takes every dependent down with it. Distribute the revocation list to each node and check it locally.

---

## 4. Data recovery

### 4.1 Design says how data is backed up and restored

For every stateful component: what is backed up, how it is restored, how loss or corruption is detected, and what happens between loss and restore. "It's derived state, rebuilt from the source of truth" is a valid answer — if the rebuild is tested and its duration known.

### 4.2 Restores verify integrity

Backups need the same protection as primary data, and restore tooling must verify checksums or signatures during the restore; unchecked signatures do nothing. Backup storage should not be writable by the same identities as the primary.

### 4.3 A checksum covers the key, not just the bytes

A content hash alone still passes when a buggy read returns the wrong object together with that object's correct checksum. Include the key or path in what is checked.

```rust
// Better: identity is part of what is verified
let digest = hash(&[key.as_bytes(), b"\0", &body].concat());
```

### 4.4 Partial restores

Split persistent data into chunks that can be checked and restored one by one, so corruption can be isolated without a full restore.

### 4.5 Practice restores

Run restore and validation exercises regularly. Restore procedures do unusual work under unfamiliar conditions and rot when unused.

---

## 5. Failover and emergency access

- **Run the failover.** Until traffic has actually moved to the secondary zone, region, or replica, it protects nothing you can rely on.
- **Playbooks reachable during the outage.** A new kill switch, failover flag, or degraded mode comes with a current runbook stored where responders can reach it when the system it helps recover is down.
- **Access control is not the only way in.** If the normal access stack (single sign-on) is down, responders still need a tested, audited emergency path, with credentials that outlive realistic outages. → security stage for how that path is controlled.

---

## Sources

- Building Secure and Reliable Systems (Google) — https://google.github.io/building-secure-and-reliable-systems/raw/toc.html
- https://aws.amazon.com/builders-library/leader-election-in-distributed-systems/ — Leader election in distributed systems
- https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/ — Using load shedding to avoid overload
