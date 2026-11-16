# Notation

Plain text only, one line per sticky note from the EventStorming board. Time runs left to right on a timeline,
top to bottom in a list. No Mermaid or other diagram languages.

## Marks

| Mark | Meaning | Example |
|---|---|---|
| `>Name` | an action someone or something asks for (a command) | `>PlaceOrder` |
| `!Name` | an event: something that happened, past tense | `!OrderPlaced` |
| `?Name` | what an actor reads before acting (a view) | `?OpenOrders` |
| `[Name]` | an outside system we depend on but do not build | `[Payments]`, `[S3]` |
| `whenever !X ──► >Y` | a rule: each time X happens, do Y | `whenever !OrderPlaced ──► >CapturePayment` |

Names: action and event use the same verb (`>PlaceOrder` → `!OrderPlaced`). A different verb means a different
fact. Plain words for everything else.

## Timeline per actor

```text
Customer ─────────────────────────────────────────────────────────────────────────►
  ?NearbyShops   >PlaceOrder        >PickUp
                      │                 │
                 !OrderPlaced      !OrderPickedUp
                      │
       whenever !OrderPlaced ──► >CapturePayment ──► [Payments] ──► !PaymentCaptured
```

Label every arrow with what moves along it when it is not obvious. Keep a diagram to about 15 lines; split by
actor or by area when it grows.

## Areas

```text
  ┌ Discovery ───────┐        ┌ Ordering ─────────┐        ┌ Payments ─┐
  │ ?NearbyShops     │──shop─►│ >PlaceOrder       │──order─►│ [Payments]│
  └──────────────────┘        │ !OrderPlaced      │◄─result─└───────────┘
                              └───────────────────┘
```

## In and out of an action

```text
?NearbyShops
  in:      user id, latitude, longitude
  out:     shops (id, name, distance), nearest first
  fails:   location unknown → the caller is asked for a location
```

## Given/When/Then

One case per line group, written with the timeline's names:

```text
>PlaceOrder — shop closed
  Given  !ShopRegistered(shop, open until 18:00)
  When   >PlaceOrder(shop, pistachio, pickup at 18:30)
  Then   rejected: shop closed at pickup time
```

## behavior.md layout

```text
# <feature> — behavior

## Who is involved
## Journeys
## Outside systems and rules
## Areas
## In and out of each action
## Edge cases            (per action: Happy path, Edge cases with status, Not handled, Invariants)
## Given/When/Then       (agreed edge cases only)
## Open questions
```

Status of an edge case: `proposed` (shown, waiting for the user) or `agreed` (the user agreed with the handling).
