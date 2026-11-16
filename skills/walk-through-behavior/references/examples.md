# Examples

Two shortened walk-throughs. The first follows the talk's ice cream app; the second is a backend consumer.

## Ice cream ordering (from the talk)

### 1. Who is involved

```text
People:            customer, shop manager, administrator (registers shops)
Outside systems:   [Payments], [Delivery]
```

Why: the talk draws one journey per actor; the administrator is easy to miss, and without them no shop exists to
order from. Payments and delivery are outside systems: "you would not write software to verify payment details."

### 2. Journey per actor

```text
Customer ──────────────────────────────────────────────────────────────────────────────────►
  it's hot   sees an ad   installs the app   ?NearbyShops   >PlaceOrder      >PickUp
                                                                 │               │
                                                            !OrderPlaced    !OrderPickedUp

Shop manager ──────────────────────────────────────────────────────────────────────────────►
                                             ?OpenOrders   >MarkReady ──► !OrderMarkedReady

Administrator ─────────────────────────────────────────────────────────────────────────────►
  >RegisterShop ──► !ShopRegistered
```

Why: business-level steps first, as in the talk. `?NearbyShops` is there because the customer cannot pick a shop
without seeing which shops are near.

### 3. Outside systems and rules

```text
whenever !OrderPlaced     ──► >CapturePayment ──► [Payments] ──► !PaymentCaptured
whenever !PaymentCaptured ──► >NotifyShop     ──► !ShopNotified
```

"Always? Immediately?" on the first rule: is payment captured at once, or only when the shop accepts? If the shop
can refuse, capturing at once means refunds; that question goes to the user.

### 4. Areas

```text
  ┌ Shop sign-up ┐   ┌ Discovery ─────┐   ┌ Ordering ──────────┐   ┌ Reviews ┐
  │ >RegisterShop│──►│ ?NearbyShops   │──►│ >PlaceOrder        │──►│         │
  └──────────────┘   └────────────────┘   │ >MarkReady >PickUp │   └─────────┘
                                          └────────┬───────────┘
                                                   ▼
                                              [Payments]
```

The talk's split: user sign-up, shop sign-up, discovery, order processing, delivery, reviews. Few arrows between
groups means the split is right.

### 5. In and out of each action

```text
?NearbyShops
  in:      user id, latitude, longitude
  out:     shops (id, name, distance), nearest first
  fails:   location unknown → ask the customer for a location
```

### 6. Happy path and edge cases

```text
Use case: Customer books a pickup

  Happy path
    The customer picks a shop and a flavour, pays, and gets a pickup code.

  Edge cases
    Shop closes before the pickup time
      Cause: shops set their own hours; a pickup can be booked hours ahead.
      Proposed handling: reject pickup times after closing minus 15 minutes.     [proposed]

    Payment succeeds but the reply is lost
      Cause: [Payments] is a remote call; the reply can time out after the charge was made.
      Proposed handling: send an idempotency key with each charge and ask [Payments] for the
      result before charging again.                                                [proposed]

  Not handled
    Two customers order the last scoop at once: shops do not track stock per scoop.

  Invariants
    A customer is charged at most once per order.

  Edge cases to add or change?
```

## Kafka snapshot consumer

### 2. Journey (the consumer process is the actor)

```text
Consumer ───────────────────────────────────────────────────────────────────────────────────►
  [Kafka] ──► !RecordsPolled ──► (window full) >WriteSnapshot ──► [S3] ──► !SnapshotWritten
                                                                              │
                         whenever !SnapshotWritten ──► >CommitOffsets ──► [Kafka] ──► !OffsetsCommitted
```

### 6. Happy path and edge cases

```text
Use case: Save a snapshot, then commit the offsets

  Happy path
    Poll records → write the snapshot to S3 → commit the offsets up to the snapshot's last offset.

  Edge cases
    Partition revoked between the snapshot write and the commit
      Cause: a rebalance can happen at any time; the write and the commit are separate calls.
      Proposed handling: skip the commit; the new owner resumes from the last committed offset.   [proposed]

    S3 write times out
      Cause: S3 is a remote call; a request can hang or fail.
      Proposed handling: retry with capped exponential backoff and jitter; stop the partition after a
      deadline (skeptic HR-io-timeout, HR-retry-backoff).                                            [proposed]

    Process restarts after the snapshot write, before the commit
      Cause: a crash or deploy can happen between any two steps.
      Proposed handling: records are read again and the snapshot is rewritten under the same key, so
      the write must be idempotent.                                                                  [proposed]

  Not handled
    The same offset committed twice: committing an offset again has no effect.

  Invariants
    Committed offset ≤ the last offset contained in the snapshot.

  Edge cases to add or change?
```

After the user agrees with the first edge case:

```text
>CommitOffsets — partition revoked after the snapshot write
  Given  !SnapshotWritten(partition 3, up to offset 120)
         !PartitionRevoked(partition 3)
  When   >CommitOffsets(partition 3)
  Then   no commit; offsets for partition 3 stay at the last commit
```
