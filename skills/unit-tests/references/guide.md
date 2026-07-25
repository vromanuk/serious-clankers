# Unit testing guide

Distilled from *Software Engineering at Google*, Chapter 12 — Unit Testing (Erik Kuefler; ed. Tom Manshreck).  
Portable rules and examples for writing and reviewing **unit tests**. Language samples follow the book (Java-style) plus short Rust notes where the same idea applies.

**Load this file** when applying the `unit-tests` skill in depth.

---

## 1. What a unit test is

Google classifies tests on two axes:

| Axis | Meaning |
|------|---------|
| **Size** | Resources the test may use (CPU, network, disk, time). **Small** tests are fast and deterministic. |
| **Scope** | How much code the test is meant to validate. |

**Unit test** = relatively **narrow scope** (e.g. one class, type, or method/function). Unit tests are *usually* small in size, but not always.

### Why unit tests dominate productivity

After preventing bugs, the main job of tests is **engineer productivity**. Unit tests help because they:

1. Tend to be **small** — fast, deterministic, runnable often for immediate feedback.  
2. Are **easy to write next to the code** — no need to stand up a whole system.  
3. Enable **high coverage** cheaply — confidence when changing code.  
4. **Fail clearly** — simple, focused tests make the cause easier to see.  
5. Serve as **documentation / examples** — how to use the unit and how it is meant to work.  

**Rule of thumb:** about **80% unit tests**, **20% broader-scoped** tests (direction, not a hard quota).

Engineers may run **thousands** of unit tests in a day (directly or via CI/local tooling). That scale only works if tests are maintainable.

---

## 2. Maintainability (the main goal)

**Maintainable tests “just work”:** after writing them, you barely think about them until they fail — and then the failure means a **real bug** with a **clear cause**.

### The anti-story (brittleness + unclarity)

An engineer adds a small feature. CI returns a wall of failures. None are real product bugs — tests encoded assumptions about **internal structure**. Fixing them burns hours; hacks make the next failure worse. Testing **drained** productivity instead of raising quality.

Two failure modes:

| Problem | Meaning |
|---------|---------|
| **Brittle** | Fails on an unrelated production change that introduced **no real bug**. |
| **Unclear** | When it fails, hard to see what was wrong, how to fix it, or what the test was for. |

Brittle suites stop being “automated” if every change needs manual test surgery. At scale, even a small % of spurious failures wastes huge time.

---

## 3. Preventing brittle tests

### 3.1 Strive for unchanging tests

**Ideal:** after a test is written, it **never changes** unless the **requirements / public behavior** of the system under test change.

Four kinds of production change:

| Change | What tests should do |
|--------|----------------------|
| **Pure refactoring** (same interface, clearer/faster internals) | **No** test edits. If tests must change, either behavior changed (not pure refactor) or tests sit at the wrong abstraction. |
| **New feature** | **Add** new tests. Existing tests stay green. Editing old tests when adding features often signals accidental behavior change or bad tests. |
| **Bug fix** | Treat like a missing case: **add** the test that would have caught the bug. Existing tests usually unchanged. |
| **Behavior change** (deliberate contract change) | **Update** affected tests. Expensive: users may depend on old behavior; coordinate. Changing a test here signals breaking an **explicit** contract; the other three cases break only **unintended** contracts if tests fail. |

**Takeaway:** after you write a test, you should not touch it while refactoring, fixing unrelated bugs, or adding features. Scale works because expansion means a **few new tests**, not rewriting the suite. Only intentional behavior breaks should force mass test edits.

### 3.2 Test via public APIs

**Most important anti-brittleness rule:** invoke the unit the way **its users** would — against its **public API**, not private helpers or serialization guts.

If tests use the system like users do, a failing test **might break a real user**. Bonus: tests become examples/docs.

#### Example 12-1 — production API (transaction)

```java
public void processTransaction(Transaction transaction) {
  if (isValid(transaction)) {
    saveToDatabase(transaction);
  }
}

private boolean isValid(Transaction t) {
  return t.getAmount() < t.getSender().getBalance();
}

private void saveToDatabase(Transaction t) {
  String s = t.getSender() + "," + t.getRecipient() + "," + t.getAmount();
  database.put(t.getId(), s);
}

public void setAccountBalance(String accountName, int balance) { /* write balance */ }
public void getAccountBalance(String accountName) { /* derive balance from stored data */ }
```

#### Example 12-2 — brittle: testing privates / internals (don’t)

```java
@Test
public void emptyAccountShouldNotBeValid() {
  assertThat(processor.isValid(newTransaction().setSender(EMPTY_ACCOUNT)))
      .isFalse();  // isValid is private implementation
}

@Test
public void shouldSaveSerializedData() {
  processor.saveToDatabase(newTransaction()
      .setId(123).setSender("me").setRecipient("you").setAmount(100));
  assertThat(database.get(123)).isEqualTo("me,you,100");  // serialization format
}
```

**Why brittle:** rename methods, extract helpers, change serialization → tests break though users never cared.

#### Example 12-3 — better: public API / user-visible state

```java
@Test
public void shouldTransferFunds() {
  processor.setAccountBalance("me", 150);
  processor.setAccountBalance("you", 20);

  processor.processTransaction(newTransaction()
      .setSender("me").setRecipient("you").setAmount(100));

  assertThat(processor.getAccountBalance("me")).isEqualTo(50);
  assertThat(processor.getAccountBalance("you")).isEqualTo(120);
}

@Test
public void shouldNotPerformInvalidTransactions() {
  processor.setAccountBalance("me", 50);
  processor.setAccountBalance("you", 20);

  processor.processTransaction(newTransaction()
      .setSender("me").setRecipient("you").setAmount(100));

  assertThat(processor.getAccountBalance("me")).isEqualTo(50);
  assertThat(processor.getAccountBalance("you")).isEqualTo(20);
}
```

Same coverage of “valid transfer / reject invalid,” without locking private structure. (“Use the front door first.”)

#### What is a “unit” / public API?

Not always language `public`. Roughly: the API exposed to parties **outside the owning team** (or to general consumers of a support library).

Rules of thumb:

1. **Helper only for one or two types** → usually **not** its own unit; test through the owning types.  
2. **Anyone can use it without consulting owners** → unit; test as users would.  
3. **Internal but general support library** → still a unit worth direct tests (some redundancy with callers is OK — fills coverage gaps).  

Language visibility (`private`/`pub(crate)`) and build visibility (Bazel, etc.) may differ from “public API” for testing.

**Rust note:** prefer testing the crate/module surface callers use (`pub` API of the unit). Avoid `#[cfg(test)]` only to poke private helpers unless that surface *is* the contract.

### 3.3 Test state, not interactions

Two ways to check results:

| Style | What you check | Brittleness |
|-------|----------------|-------------|
| **State** | What the system looks like after the call | Lower — care about outcome |
| **Interaction** | Sequence of calls on collaborators | Higher — care about *how* |

Many tests mix both; prefer state when you can.

#### Example 12-4 — brittle interaction test

```java
@Test
public void shouldWriteToDatabase() {
  accounts.createUser("foobar");
  verify(database).put("foobar");  // exact collaborator call
}
```

Fails wrongly if:

- Record is written then deleted (bug) but `put` still happened → **false pass**.  
- Refactor writes an equivalent record via a slightly different API → **false fail**.  

#### Example 12-5 — state instead

```java
@Test
public void shouldCreateUsers() {
  accounts.createUser("foobar");
  assertThat(accounts.getUser("foobar")).isNotNull();
}
```

States what you care about: after the action, the user exists.

**Mocking:** frameworks that record/verify every call push you toward brittle interaction tests. Prefer **real objects** when they are fast and deterministic. Reserve mocks for slow/nondeterministic/external boundaries — and still assert **outcomes** when possible.

**Rust note:** avoid assert-only “was this trait method called N times” as the sole check when you can assert returned state or stored data.

---

## 4. Writing clear tests

Unclear tests hurt as much as brittle ones: failures don’t teach, and readers can’t tell intent.

### 4.1 Complete and concise

| Property | Meaning |
|----------|---------|
| **Complete** | Contains all information needed to understand the behavior under test (relevant inputs, context, expected outcome). |
| **Concise** | Does **not** contain irrelevant information that hides the point. |

#### Example 12-6 — incomplete + cluttered (bad)

A test that buries the important values in huge setup, or omits the expected outcome in the name/body so readers must reverse-engineer.

#### Example 12-7 — complete and concise (good)

Only the accounts, amounts, and expected balances that matter for **this** behavior; assertion states the outcome clearly.

**Duplication vs cleverness:** humans miss bugs in concatenated strings and clever helpers. Duplicating a base URL in a test can be worth it so the expected value is visible (see DAMP later).

### 4.2 Test behaviors, not methods

Don’t write “one test class per production class, one test method per production method.” That couples tests to structure and produces shallow tests.

Instead: **features = collections of behaviors**. Each test encodes **one behavior** (given a situation, when an action happens, then an outcome).

#### Example 12-8–10 — method-driven vs behavior-driven

**Method-driven (weaker):** `testProcessTransaction`, `testIsValid`, … maps 1:1 to methods, including privates.

**Behavior-driven (stronger):**  

- `shouldTransferFundsWhenBalanceSufficient`  
- `shouldRejectTransferWhenBalanceInsufficient`  

Multiple behaviors may exercise the same method; that’s fine.

### 4.3 Structure tests to emphasize behaviors

Use a clear narrative (names vary; idea is the same):

```text
Given  — arrange world
When   — act
Then   — assert
```

(Also called arrange / act / assert.)

#### Example 12-11 — well-structured

```text
// Given
set balances…

// When
processTransaction(…)

// Then
assert balances…
```

#### Example 12-12 — multiple when/then

If one test needs several acts, separate blocks clearly — or split into multiple behavior tests if they’re independent stories.

### 4.4 Name tests after the behavior

Names should read like a specification sentence.

Patterns (book samples):

- Nested / descriptive: `transferFunds / when balance sufficient / updates both accounts`  
- Method-style still OK if the **name is the behavior**: `shouldUpdateBothAccountsWhenBalanceSufficient`

Avoid: `test1`, `testProcessTransaction`, `works`.

### 4.5 Don’t put logic in tests

**No** conditionals, loops, or clever helpers that re-implement production logic in the test. Stick to **straight-line** code.

#### Example 12-15–16 — logic conceals bugs

If the test builds expected values with the same formula as production, both can be wrong and the test still passes. Hard-coded expected outcomes reveal the bug.

**Tolerate duplication** when it makes the expected result obvious.

### 4.6 Write clear failure messages

Ideal: engineer diagnoses from the **failure message** alone.

Bad: `Test failed: account is closed` (expected closed or actual closed?)  

Better: expected state, actual state, relevant fields:

```text
Expected an account in state CLOSED, but got account:
  <{name: "my-account", state: "OPEN"}>
```

#### Example 12-17 — assertion libraries (Java)

```java
Set<String> colors = ImmutableSet.of("red", "green", "blue");
assertTrue(colors.contains("orange"));  // weak message
assertThat(colors).contains("orange");  // Truth: richer failure
```

#### Example 12-18 — Go-style explicit messages

Same idea: assertions that print expected vs actual with context.

**Rust note:** prefer `assert_eq!(got, expected)` / `pretty_assertions` / custom messages over bare `assert!(cond)`.

---

## 5. DAMP over DRY in tests

Production code often optimizes **DRY**. Test code optimizes **reader clarity**.

**DAMP** = Descriptive And Meaningful Phrases (favor readability and local clarity).  
**DRY** = Don’t Repeat Yourself.

### Too DRY (Example 12-19)

Heavy shared setup + loops hide which values matter for **this** behavior; failures are opaque.

### DAMP (Example 12-20)

Each test shows the important inputs and expectations even if similar lines repeat across tests.

### Sharing code carefully

Sharing **can** help if it keeps tests descriptive:

| Pattern | Guidance |
|---------|----------|
| **Shared constants with vague names** (Ex 12-21) | Bad — readers don’t know role of `VALUE_A`. |
| **Helper factories with explicit overrides** (Ex 12-22) | Better — defaults for uninteresting fields; test sets only fields it cares about. Optionally **slightly randomize** defaults for unspecified fields so accidental equality is rarer and hardcoding defaults is harder. |
| **setUp that tests silently depend on** (Ex 12-23) | Risky — important given state is invisible in the test body. |
| **Override in the test** (Ex 12-24) | Better — make the relevant given state local or obviously overridden. |

**Conceptually simple tests** (Ex 12-25): a reader should reconstruct the story without spelunking setup hierarchies.

### Test infrastructure (cross-suite)

Code shared across many suites is **infrastructure**, more like production:

- Many dependents → hard to change.  
- Treat as its own product **with its own tests**.  
- Prefer standard org-wide libraries early (e.g. Google standardized on Mockito for Java mocks).  

---

## 6. Size, scope, and the 80/20 mix

| | Unit (typical) | Broader |
|--|----------------|--------|
| Scope | One function/type/module | Multi-module, real DB/network |
| Size | Prefer small: fast, deterministic | Fewer, slower |
| Count | Most of the suite (~80%) | Minority (~20%) |

Unit tests need not always be “smallest possible size,” but narrow scope + small size is the sweet spot for daily workflow.

---

## 7. Review checklist (for humans and agents)

When reviewing or writing unit tests, ask:

1. **Unchanging?** Would a pure internal refactor force this test to change? If yes → too low-level.  
2. **Public API?** Does it call the unit like a user of that unit?  
3. **State vs interaction?** Are we asserting outcomes/state, or only mock call sequences?  
4. **One behavior?** Name and body about a single story?  
5. **Complete + concise?** All needed context present; no clutter?  
6. **Straight-line?** No production-duplicating logic in the test?  
7. **Failure message?** Expected vs actual clear?  
8. **DAMP?** Shared setup still leaves important values obvious?  
9. **New behavior covered?** Feature/bugfix added tests without rewriting the world?  

---

## 8. Mapping to this repo

| Idea | Where else |
|------|------------|
| Thinking code vs shell (what is easy to unit-test) | `skeptic/references/examples/pure-core.md`, skeptic-testability |
| Property / snapshot **when** | `skeptic/references/testing.md` (still obey public-API + unchanging rules) |
| Hard ban: new thinking behavior without tests | `skeptic/references/hard-rules.md` → `HR-new-behavior-no-test` |

---

## 9. TL;DRs (from the chapter)

1. Strive for **unchanging** tests.  
2. Test via **public APIs**.  
3. Test **state**, not interactions.  
4. Make tests **complete and concise**.  
5. Test **behaviors**, not methods.  
6. **Structure** tests to emphasize behaviors (Given/When/Then).  
7. **Name** tests after the behavior.  
8. Don’t put **logic** in tests.  
9. Write **clear failure messages**.  
10. Follow **DAMP over DRY** when sharing test code.  

---

## Sources

- Erik Kuefler, “Unit Testing,” in *Software Engineering at Google* (O’Reilly), Chapter 12; ed. Tom Manshreck.  
- Related: Testing on the Toilet — test behaviors not methods; Dan North — BDD intro.  
- Flaky vs brittle: brittle = fails on harmless prod change; flaky = nondeterministic without prod change.  
