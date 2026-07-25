# Exemplars — thinking code vs shell (plain words)

Use plain names. Do not require coined labels in findings.

## Good shape

```rust
// Thinking: pure decision
enum Plan { DoNothing, Update(Customer), UpdateAndEmail(Customer, String) }

fn update_customer(existing: &Customer, new: &Customer) -> Plan {
    // no db, no email, no Instant::now
    ...
}

// Shell: effects only
fn handle(db: &Db, mail: &Mail, id: Id) {
    let existing = db.read(id);
    let new = db.read_incoming(id);
    match update_customer(&existing, &new) {
        Plan::DoNothing => {}
        Plan::Update(c) => db.write(c),
        Plan::UpdateAndEmail(c, msg) => {
            db.write(c);
            mail.send(msg);
        }
    }
}
```

## Bad shape

```rust
async fn update_customer(...) {
    let existing = db.read().await;      // IO
    if existing == new { return; }
    db.write(new).await;                 // rule buried between IO
    if needs_email { mail.send().await; }
}
```

Nothing useful to unit-test without mocks of the world.

## Time

```rust
// Good: now is a parameter
fn expired(now: Instant, deadline: Instant) -> bool { now >= deadline }

// Bad: now() inside the core
fn expired(deadline: Instant) -> bool { Instant::now() >= deadline }
```
