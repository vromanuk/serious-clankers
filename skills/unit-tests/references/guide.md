# Unit testing guide

Distilled from *Software Engineering at Google*, Chapter 12 — Unit Testing (Erik Kuefler; ed. Tom Manshreck).  
Portable rules for writing and reviewing **unit tests**.

**Language of the samples:** most code blocks are **Java** (as in the book), with a few Go/JS/Python snippets and occasional Rust notes. Treat them as **illustrations of the idea**, not as a Java-only rulebook. The same practices apply in Rust, Go, TypeScript, or any other stack — restate them in the project’s language when you write or review tests.

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

Neither complete nor concise: noise in the constructor, and the real inputs hide inside a helper.

```java
@Test
public void shouldPerformAddition() {
  Calculator calculator = new Calculator(new RoundingStrategy(),
      "unused", ENABLE_COSINE_FEATURE, 0.01, calculusEngine, false);
  int result = calculator.calculate(newTestCalculation());
  assertThat(result).isEqualTo(5); // Where did this number come from?
}
```

| Problem | What’s wrong |
|---------|----------------|
| **Not concise** | Constructor args (`"unused"`, cosine flag, tolerance, engine, …) are irrelevant to addition and distract the reader. |
| **Not complete** | The meaningful inputs (e.g. 2 + 3) are buried in `newTestCalculation()`; expected `5` has no visible link to those inputs. |

#### Example 12-7 — complete and concise (good)

Same behavior: hide *irrelevant* construction; put *behavior-critical* values in the test body.

```java
@Test
public void shouldPerformAddition() {
  Calculator calculator = newCalculator();  // hides boring defaults
  int result = calculator.calculate(newCalculation(2, Operation.PLUS, 3));
  assertThat(result).isEqualTo(5);  // 2 + 3 → 5 is obvious
}
```

| Fix | How |
|-----|-----|
| **More complete** | `2`, `PLUS`, `3` and expected `5` are all in the body — no reverse-engineering. |
| **More concise** | `newCalculator()` drops constructor noise that does not affect this behavior. |

**Rule:** the body should hold everything needed to understand the result, and nothing that only distracts. Helpers are good for **uninteresting** setup; they are bad when they hide **what makes this test this test**.

**Duplication vs cleverness:** humans miss bugs in concatenated strings and clever helpers. Duplicating a base URL in a test can be worth it so the expected value is visible (see DAMP later).

### 4.2 Test behaviors, not methods

Don’t map “one production method → one test method.” Methods grow; tests become bags of unrelated checks (and violate unchanging tests when people pile on).

A **behavior** is a guarantee: given a state, when an action happens, then an outcome. Method ↔ behavior is many-to-many.

#### Example 12-8 — production (two behaviors in one method)

```java
public void displayTransactionResults(User user, Transaction transaction) {
  ui.showMessage("You bought a " + transaction.getItemName());
  if (user.getBalance() < LOW_BALANCE_THRESHOLD) {
    ui.showMessage("Warning: your balance is low!");
  }
}
```

#### Example 12-9 — method-driven (weaker)

One test grows as the method gains features:

```java
@Test
public void testDisplayTransactionResults() {
  transactionProcessor.displayTransactionResults(
      newUserWithBalance(LOW_BALANCE_THRESHOLD.plus(dollars(2))),
      new Transaction("Some Item", dollars(3)));

  assertThat(ui.getText()).contains("You bought a Some Item");
  assertThat(ui.getText()).contains("your balance is low");
}
```

#### Example 12-10 — behavior-driven (stronger)

Split by story; extra boilerplate is worth it:

```java
@Test
public void displayTransactionResults_showsItemName() {
  transactionProcessor.displayTransactionResults(
      new User(), new Transaction("Some Item"));
  assertThat(ui.getText()).contains("You bought a Some Item");
}

@Test
public void displayTransactionResults_showsLowBalanceWarning() {
  transactionProcessor.displayTransactionResults(
      newUserWithBalance(LOW_BALANCE_THRESHOLD.plus(dollars(2))),
      new Transaction("Some Item", dollars(3)));
  assertThat(ui.getText()).contains("your balance is low");
}
```

### 4.3 Structure tests to emphasize behaviors

Make **Given / When / Then** (arrange / act / assert) explicit — comments and whitespace, or framework support.

#### Example 12-11 — well-structured

```java
@Test
public void transferFundsShouldMoveMoneyBetweenAccounts() {
  // Given two accounts with initial balances of $150 and $20
  Account account1 = newAccountWithBalance(usd(150));
  Account account2 = newAccountWithBalance(usd(20));

  // When transferring $100 from the first to the second account
  bank.transferFunds(account1, account2, usd(100));

  // Then the new account balances should reflect the transfer
  assertThat(account1.getBalance()).isEqualTo(usd(50));
  assertThat(account2.getBalance()).isEqualTo(usd(120));
}
```

Read at three levels: method name → given/when/then comments → code.  
Don’t interleave many acts and asserts unless you mean a multi-step **single** behavior.

#### Example 12-12 — alternating when/then (one multi-step behavior)

```java
@Test
public void shouldTimeOutConnections() {
  // Given two users
  User user1 = newUser();
  User user2 = newUser();

  // And an empty connection pool with a 10-minute timeout
  Pool pool = newPool(Duration.minutes(10));

  // When connecting both users to the pool
  pool.connect(user1);
  pool.connect(user2);

  // Then the pool should have two connections
  assertThat(pool.getConnections()).hasSize(2);

  // When waiting for 20 minutes
  clock.advance(Duration.minutes(20));

  // Then the pool should have no connections
  assertThat(pool.getConnections()).isEmpty();

  // And each user should be disconnected
  assertThat(user1.isConnected()).isFalse();
  assertThat(user2.isConnected()).isFalse();
}
```

Most unit tests need only one when + one then. If you need “and” in the **name**, you might be testing multiple behaviors.

### 4.4 Name tests after the behavior

Names show up first in failure reports. Describe **action + expected outcome** (and setup when needed).

#### Example 12-13 — nested naming (e.g. Jasmine)

```javascript
describe("multiplication", function() {
  describe("with a positive number", function() {
    var positiveNumber = 10;
    it("is positive with another positive number", function() {
      expect(positiveNumber * 10).toBeGreaterThan(0);
    });
    it("is negative with a negative number", function() {
      expect(positiveNumber * -10).toBeLessThan(0);
    });
  });
  // ...
});
```

#### Example 12-14 — method-name patterns

```text
multiplyingTwoPositiveNumbersShouldReturnAPositiveNumber
multiply_positiveAndNegative_returnsNegative
divide_byZero_throwsException
```

Verbose is OK: tests aren’t called from production. Trick: start with **should…** so class + name read as a sentence  
(`BankAccount` + `shouldNotAllowWithdrawalsWhenBalanceIsEmpty`).

Avoid: `test1`, `testProcessTransaction`, `works`. Word **and** in a name → maybe split tests.

### 4.5 Don’t put logic in tests

No operators/loops/conditionals that force mental execution. If a test needs its own tests, something is wrong. Prefer straight-line code; tolerate duplication for clarity.

#### Example 12-15 — logic conceals a bug

```java
@Test
public void shouldNavigateToAlbumsPage() {
  String baseUrl = "http://photos.google.com/";
  Navigator nav = new Navigator(baseUrl);
  nav.goToAlbumPage();
  assertThat(nav.getCurrentUrl()).isEqualTo(baseUrl + "/albums");
}
```

One concatenation — still easy to miss a double-slash bug shared with production.

#### Example 12-16 — no logic; bug is visible

```java
@Test
public void shouldNavigateToPhotosPage() {
  Navigator nav = new Navigator("http://photos.google.com/");
  nav.goToPhotosPage();
  assertThat(nav.getCurrentUrl())
      .isEqualTo("http://photos.google.com//albums"); // Oops! double slash
}
```

Duplicating the base URL is worth it so expected strings are readable. Loops/conditionals in tests hide bugs even more.

### 4.6 Write clear failure messages

Ideal: diagnose from the log without opening the test. Include desired outcome, actual outcome, relevant params.

**Bad:** `Test failed: account is closed` (expected closed, or was closed?)  

**Better:**

```text
Expected an account in state CLOSED, but got account:
  <{name: "my-account", state: "OPEN"}>
```

#### Example 12-17 — assertion libraries (Java)

```java
Set<String> colors = ImmutableSet.of("red", "green", "blue");
assertTrue(colors.contains("orange"));  // "expected true but was false"
assertThat(colors).contains("orange");  // Truth: subject in the message
// e.g. <[red, green, blue]> should have contained <orange>
```

#### Example 12-18 — Go-style explicit messages

```go
result := Add(2, 3)
if result != 5 {
  t.Errorf("Add(2, 3) = %v, want %v", result, 5)
}
```

**Rust note:** prefer `assert_eq!(got, expected)` / `pretty_assertions` / custom messages over bare `assert!(cond)`.

---

## 5. DAMP over DRY in tests

Production often optimizes **DRY**. Tests optimize **clarity and stability** (they should break when behavior changes, not when you extract a helper in prod).

**DAMP** = Descriptive And Meaningful Phrases.  
**DRY** = Don’t Repeat Yourself.  
DAMP **complements** DRY: helpers are fine when they make tests *clearer*, not only shorter.

### Example 12-19 — too DRY (bad)

```java
@Test
public void shouldAllowMultipleUsers() {
  List<User> users = createUsers(false, false);
  Forum forum = createForumAndRegisterUsers(users);
  validateForumAndUsers(forum, users);
}

@Test
public void shouldNotAllowBannedUsers() {
  List<User> users = createUsers(true);
  Forum forum = createForumAndRegisterUsers(users);
  validateForumAndUsers(forum, users);
}

// helpers with loops/conditionals hide important details and can hide bugs
private static List<User> createUsers(boolean... banned) { /* loop + setState */ }
private static Forum createForumAndRegisterUsers(List<User> users) { /* loop + catch */ }
private static void validateForumAndUsers(Forum forum, List<User> users) {
  // e.g. wrong equality vs BANNED state buried in a loop
}
```

Bodies are short but **incomplete**; logic in helpers is hard to verify at a glance.

### Example 12-20 — DAMP (good)

```java
@Test
public void shouldAllowMultipleUsers() {
  User user1 = newUser().setState(State.NORMAL).build();
  User user2 = newUser().setState(State.NORMAL).build();

  Forum forum = new Forum();
  forum.register(user1);
  forum.register(user2);

  assertThat(forum.hasRegisteredUser(user1)).isTrue();
  assertThat(forum.hasRegisteredUser(user2)).isTrue();
}

@Test
public void shouldNotRegisterBannedUsers() {
  User user = newUser().setState(State.BANNED).build();

  Forum forum = new Forum();
  try {
    forum.register(user);
  } catch (BannedUserException ignored) {}

  assertThat(forum.hasRegisteredUser(user)).isFalse();
}
```

More lines, but each test is meaningful **without leaving the body**.

### Example 12-21 — shared values, ambiguous names (bad)

```java
private static final Account ACCOUNT_1 = Account.newBuilder()
    .setState(AccountState.OPEN).setBalance(50).build();
private static final Account ACCOUNT_2 = Account.newBuilder()
    .setState(AccountState.CLOSED).setBalance(0).build();
private static final Item ITEM = Item.newBuilder()
    .setName("Cheeseburger").setPrice(100).build();

@Test
public void canBuyItem_returnsFalseForClosedAccounts() {
  assertThat(store.canBuyItem(ITEM, ACCOUNT_1)).isFalse(); // which is closed?
}

@Test
public void canBuyItem_returnsFalseWhenBalanceInsufficient() {
  assertThat(store.canBuyItem(ITEM, ACCOUNT_2)).isFalse();
}
```

Even with good **test names**, readers scroll for definitions; reuse encourages wrong constant for the scenario. Prefer descriptive names **or** (better) build values in the test / via helpers.

### Example 12-22 — factory helpers with defaults (good)

```python
def newContact(firstName="Grace", lastName="Hopper", phoneNumber="555-123-4567"):
  return Contact(firstName, lastName, phoneNumber)

def test_fullNameShouldCombineFirstAndLastNames(self):
  contact = newContact(firstName="Ada", lastName="Lovelace")
  self.assertEqual(contact.fullName(), "Ada Lovelace")
```

```java
private static Contact.Builder newContact() {
  return Contact.newBuilder()
      .setFirstName("Grace")
      .setLastName("Hopper")
      .setPhoneNumber("555-123-4567");
}

@Test
public void fullNameShouldCombineFirstAndLastNames() {
  Contact contact = newContact()
      .setFirstName("Ada")
      .setLastName("Lovelace")
      .build();
  assertThat(contact.getFullName()).isEqualTo("Ada Lovelace");
}
```

Optional: slightly **randomize** defaults for fields tests don’t set, so accidental equality and hardcoded defaults are harder.

### Example 12-23 — depend on setUp values (incomplete)

```java
@Before
public void setUp() {
  nameService = new NameService();
  nameService.set("user1", "Donald Knuth");
  userStore = new UserStore(nameService);
}

// ... hundreds of lines later ...

@Test
public void shouldReturnNameFromService() {
  UserDetails user = userStore.get("user1");
  assertThat(user.getName()).isEqualTo("Donald Knuth"); // where from?
}
```

### Example 12-24 — override important values in the test (better)

```java
@Before
public void setUp() {
  nameService = new NameService();
  nameService.set("user1", "Donald Knuth");
  userStore = new UserStore(nameService);
}

@Test
public void shouldReturnNameFromService() {
  nameService.set("user1", "Margaret Hamilton");
  UserDetails user = userStore.get("user1");
  assertThat(user.getName()).isEqualTo("Margaret Hamilton");
}
```

Setup is fine for boring construction; tests that care about **particular** values should state them.

### Example 12-25 — focused validation helper (OK when one fact)

General `validateEverything()` at the end of every test blurs behaviors. A helper that asserts **one** conceptual fact — especially if it needs a loop — can still help:

```java
private void assertUserHasAccessToAccount(User user, Account account) {
  for (long userId : account.getUsersWithAccess()) {
    if (user.getId() == userId) {
      return;
    }
  }
  fail(user.getName() + " cannot access " + account.getName());
}
```

### Test infrastructure (cross-suite)

Code shared across many suites is **infrastructure**, more like production:

- Many dependents → hard to change.  
- Treat as its own product **with its own tests**.  
- Prefer standard org-wide libraries early (e.g. one mocking framework for the org).  

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
