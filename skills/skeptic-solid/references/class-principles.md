# SOLID class principles, applied to Rust

Reference for reviewing how types, traits, and modules are shaped so that the next likely change is **small and local**. Examples are Rust; Noback's book uses PHP classes and interfaces — read "class" as a struct with its `impl`, or a module, and "interface" as a trait.

---

## 0. How to use these principles

### 0.1 Guidelines, judged by the next change

The principles prepare a codebase for change: "You want these changes to be local, not global, and small, not big." They are guidelines, not laws — each "brings some room for discussion", and "not all of them can or should be applied all the time".

So every finding must name the **change pressure**: which group of people asks for changes, which variant keeps being added, which caller is confused, which volatile dependency leaks in. "Violates SRP" alone is not a finding.

### 0.2 Symptoms of a design that resists change

Martin's four symptoms, all caused by poor dependencies between modules:

- **Rigidity** — "Every change causes a cascade of subsequent changes in dependent modules."
- **Fragility** — changes break things in places "that have no conceptual relationship with the area that was changed."
- **Immobility** — the code cannot be reused, "And so the software is simply rewritten instead of reused."
- **Viscosity** — the design-preserving change is harder than the hack, so people take the hack.

### 0.3 The pack's defaults win where naive SOLID would conflict

- **Deep units over many tiny ones** (complexity stage): splitting a clear unit into several small types that must be read together is a red flag, not SRP.
- **An enum when the crate owns every variant; a trait only for an open set, another crate's implementations, or a real boundary** (architecture stage). A trait with one production implementation plus a test fake is a test seam problem, not DIP.
- **Collaborators passed in at the app edge** (architecture stage, "Passed in, not built inside") — DIP builds on it; do not repeat it.

Noback agrees with these defaults: most types should get no interface at all (§ 5.4).

---

## 1. Single responsibility

> "A class should have one, and only one, reason to change." — "each responsibility is also a reason to change."

Martin later clarified that a reason to change is **a group of people**: "those changes can only originate from a single person, or rather, a single tightly coupled group of people representing a single narrowly defined business function." Stated as cohesion: "Gather together the things that change for the same reasons. Separate those things that change for different reasons." That is why "business rules should not know the database schema", and why results are not formatted in the modules that compute them.

### 1.1 What to flag

- A type or module that answers to different groups: pricing rules next to SQL next to report wording; one `impl` mixing domain math, `sqlx` queries, and formatting.
- Noback's symptoms list: "many instance variables", "many public methods", "Each method of the class uses different instance variables", "Specific tasks are delegated to private methods."
- A constructor that keeps growing: "When a class has many responsibilities, it is likely to have many dependencies too."
- A test that must set up the translator, the templates, and the mailer just to check the subject line.

### 1.2 Example

```rust
// Worse: building the message and sending it change for different reasons
// (copy and templates vs transport), and every test needs all three collaborators.
pub struct ConfirmationMailer { templates: Templates, translations: Translations, transport: SmtpTransport }

impl ConfirmationMailer {
    pub async fn send_to(&self, user: &User) -> Result<(), MailError> {
        let subject = self.translations.get("Confirm your mail address");
        let body = self.templates.render("confirmation_mail", user.confirmation_code())?;
        self.transport.send(Message::new(subject, body, user.email())).await
    }
}

// Better: building the message is its own unit (pure, tested with fixed inputs);
// the mailer only sends.
pub struct ConfirmationMail { templates: Templates, translations: Translations }

impl ConfirmationMail {
    pub fn build_for(&self, user: &User) -> Result<Message, TemplateError> { /* … */ }
}

pub struct ConfirmationMailer { mail: ConfirmationMail, transport: SmtpTransport }
```

### 1.3 When not to apply

- **Code that changes together stays together.** Splitting content from format often creates "artificial seams": a new field then has to be copied through several mirror structs. Before accepting a split, name the different reasons to change.
- **Fits in a reader's head.** Sometimes bundling makes code easier to reason about than an artificial split; sometimes one responsibility reads better as several steps. Judge by how easily the next reader understands it.
- A split that yields pieces readable only together is conjoined (complexity stage), not single-responsibility.

---

## 2. Open/closed

> "You should be able to extend a class's behavior without modifying it." — "Extension of a class means that you can influence its behavior from the outside and leave the class, or the entire class hierarchy, untouched."

When requirements change, "only part of the existing code is wrong. Much of the existing code is still right" — so a new case should not force edits to the code that is still right.

### 2.1 What to flag

Noback's list of signs that a class is not open for extension:

- "It contains conditions to determine a strategy."
- "Conditions using the same variables or constants are recurring inside the class or related classes."
- "The class contains hard-coded references to other classes or class names."
- "Inside the class, objects are being created using the new operator."

Plus:

- **Moving the switch into a factory only moves the problem.** A factory with a hard-coded list is still closed; make it take its creators from outside.
- **Temporal coupling**: a trait with `prepare()` that callers must call before `encode()`. Variant-specific steps belong inside each implementation.
- **Mutable services**: registration methods on a shared service (`&mut self` on something shared as `Arc<…>`). Prefer a builder that produces an immutable value.

### 2.2 Example

```rust
// Worse (when users keep adding formats): the same choice in several functions;
// each new format edits every one of them, and a missed arm is a bug
fn encode(format: &str, data: &Data) -> Result<Vec<u8>, EncodeError> {
    match format { "json" => /* … */, "xml" => /* … */, _ => Err(EncodeError::Unknown) }
}
fn content_type(format: &str) -> &'static str {
    match format { "json" => "application/json", "xml" => "application/xml", _ => "" }
}

// Better (open set): one trait, implementations registered at the app edge
pub trait Encoder {
    fn content_type(&self) -> &'static str;
    fn encode(&self, data: &Data) -> Result<Vec<u8>, EncodeError>; // prepares its own data
}
pub struct Encoders(HashMap<&'static str, Box<dyn Encoder + Send + Sync>>); // built in main
```

### 2.3 When not to apply

- **A closed set the crate owns is an enum**, matched in one place or behind methods on the enum. Adding a variant is a deliberate change, and the compiler finds every match. That is the architecture default and it is fine.
- **Simple code is the cheapest thing to change.** Do not add a strategy, decorator, or plugin hook just to avoid editing a small function with one caller and good tests; all added code is cost.
- Open/closed matters most for code others cannot edit — reusable crates and published libraries.

---

## 3. Liskov substitution

> "Derived classes must be substitutable for their base classes."

In Rust there is no inheritance, but every trait implementation is a subtype of its trait: "A program that uses an interface must not be confused by an implementation of that interface." In contract terms: preconditions no stronger, postconditions no weaker — "expect no more and provide no less."

Noback's summary of a good substitute:

- "Provides an implementation for all the methods of the base class."
- "Returns the type of things the base class prescribes (or more specific types)."
- "Doesn't put extra constraints on arguments for methods."

### 3.1 What to flag

- `unimplemented!()`, `todo!()`, `panic!("not supported")`, or an `Err(Unsupported)` the trait's contract does not allow.
- A silent no-op (`Ok(())`) in place of a significant operation other code relies on.
- An implementation that rejects inputs the trait accepts (it matches an enum argument and errors on variants the trait allows).
- Callers that check which implementation they have: `downcast_ref`, `type_id`, `is_special()` / `supports_x()` flags before each call. "If an implementation confuses the user of the base type, then if/switch statements will proliferate."
- Phrases like "not every X is a Y" or "only works with …" in docs, comments, or error messages — "They usually indicate that there is something wrong with the type hierarchy".
- A shared trait with several implementations and no written contract (ordering, idempotency, errors). Run one contract test against every implementation.

### 3.2 Example

```rust
// Worse: one implementation cannot honor the contract
pub trait File { fn rename(&mut self, name: &str) -> io::Result<()>; fn change_owner(&mut self, owner: Uid) -> io::Result<()>; }
impl File for DropboxFile {
    fn rename(&mut self, name: &str) -> io::Result<()> { /* … */ }
    fn change_owner(&mut self, _: Uid) -> io::Result<()> { unimplemented!("Dropbox has no owners") }
}

// Better: the capability is its own trait; callers that need it ask for it
pub trait File { fn rename(&mut self, name: &str) -> io::Result<()>; }
pub trait FileWithOwner: File { fn change_owner(&mut self, owner: Uid) -> io::Result<()>; }
// DropboxFile: File;  LocalFile: FileWithOwner
```

### 3.3 Fixes, in order

1. Add the missing method to the trait so every implementation returns something sensible.
2. Split off a more specific trait and require it where it is needed.
3. **Remove the abstraction** if only one implementation can really exist: "we could just as well leave out the inappropriate abstraction".
4. A runtime type check is the last resort.

Keep abstractions as general as their purpose and no more: "don't try to fit every possible specific thing in the world into your abstraction."

---

## 4. Interface segregation

> "Make fine-grained interfaces that are client specific." — Martin: "Clients should not be forced to depend on methods they do not use."

### 4.1 What to flag

- **Several kinds of client, one interface.** Call sites cluster into groups that use separate subsets of a trait, or a new method is added for one new caller.
- **Configure and use on one type.** Runtime code receives something it can also mutate: a handler gets `&mut Registry` where a read-only lookup would do.
- **Clients take the concrete type.** It does not matter that a type has many methods "as long as all parts of the application depend only on one small, client-specific part of the public API". A function taking `&ServiceContainer` instead of `&impl Lookup` depends on everything.
- **Making an internal method `pub` for one sibling.** It joins every client's implicit interface and becomes a backward-compatibility trap. Use `pub(crate)` or hand that caller a separate role trait.
- **A trait that copies every public method of one struct** ("header interface") with no reference to its callers, or one that exposes internals (`connection()`, `cache()`, `inner()`).
- **Test fakes full of `unimplemented!()`** — the trait is wider than its callers need.

### 4.2 Example

```rust
// Worse: every reader can also register and reset
pub trait Registry { fn get(&self, name: &str) -> Option<Arc<Handler>>; fn register(&mut self, name: &str, handler: Handler); fn reset(&mut self); }

// Better: roles from how clients use it; one type may implement both
pub trait HandlerLookup { fn get(&self, name: &str) -> Option<Arc<Handler>>; }
pub trait HandlerRegistration { fn register(&mut self, name: &str, handler: Handler); }
```

### 4.3 When not to apply

- Split by **kind** of client, not one trait per caller; "care must be taken not to overdo it" — hundreds of narrow interfaces are their own mess.
- Prefer small, role-based types from the start; segregation is mainly the way out of an existing large type. If the type itself can be split, split it.

---

## 5. Dependency inversion

> "Depend on abstractions, not on concretions." — "High-level modules should not depend upon low-level modules. Both should depend upon abstractions."

High-level policy cares little about the details that implement it, and concrete details change far more often than abstractions. Martin restates it as: dependencies, "especially those that cross architectural boundaries, point towards high level abstractions, not low level details."

### 5.1 What to flag

- **Policy depending on mechanism.** Domain or business code that takes a connection pool, runs SQL, or touches files directly.
- **Dependencies named after the technology** (`DbConnection`, `S3Client`) where the consumer needs a role (`UserProvider`, `BlobStore`). Ask: "what does this code really need?"
- **Abstractions that leak the mechanism.** A role trait with `table_name()`, `bucket()`, or `sql()` mixes levels and breaks substitution for any other implementation.
- **The trait lives on the wrong side.** A trait the domain uses declared in the infrastructure crate, so the domain must depend on infrastructure to name it. "Interfaces are very often included in the package that uses them, rather than in the package that implements them."
- **Vendor lock-in in reusable code.** A library's public API takes or returns one framework's types where a small trait of its own would do.

### 5.2 Example

```rust
// Worse: the authentication policy knows the storage mechanism
pub struct Authentication { pool: PgPool }
impl Authentication {
    pub async fn check(&self, name: &str, password: &str) -> Result<bool, AuthError> {
        let row = sqlx::query!("SELECT pw_hash FROM users WHERE name = $1", name).fetch_optional(&self.pool).await?;
        /* … */
    }
}

// Better: the policy owns the abstraction it needs, named for what it needs;
// the Postgres implementation lives in an infrastructure module and depends on it
pub trait UserProvider { async fn find_user(&self, name: &str) -> Result<Option<User>, LookupError>; }
pub struct Authentication<P: UserProvider> { users: P }
```

### 5.3 Third-party code

- Define your own **minimal** trait plus an adapter when the library's API does not fit your design, or you are not sure it is safe to rely on: switching libraries then rewrites only the adapter.
- Do **not** invert dependencies on single-purpose libraries you use as building blocks — Noback lists assertions, reflection, mapping, encoding/decoding, inflection; in Rust, `serde`, `std` collections, and similar. Martin: "if you have tried and true modules that are concrete, but not volatile, depending upon them is not so bad."
- "You can always add an interface later."

### 5.4 When to add a trait at all — Noback's checklist

Add an interface:

1. When not all public methods are meant for ordinary clients (publish a trimmed one).
2. When the type does I/O — network, filesystem, randomness, the clock.
3. When the type depends on third-party code you want to isolate.
4. When you treat several specific things the same way (an open set of implementations).
5. When users will want to replace one piece of a composed object graph.

"For everything else: stick to a final class" — especially entities, value objects, stateful objects, and business calculations. In Rust: a concrete struct with private fields.

This matches the pack's rule: an enum for variants the crate owns, a trait for an open set or a real boundary, and one `#[cfg(test)]` seam rather than a trait whose only second implementation is a mock. North puts it more bluntly: invert a dependency only when "there might be multiple ways of providing it" and the wiring matters on its own — "mostly all you ever need is a main method." Flag `trait FooService` + `struct FooServiceImpl` pairs.

---

## Sources

- Principles of Package Design (Matthias Noback, Apress, 2018) — https://doi.org/10.1007/978-1-4842-4119-6
- The Principles of OOD (Robert C. Martin) — http://butunclebob.com/ArticleS.UncleBob.PrinciplesOfOod
- Design Principles and Design Patterns (Robert C. Martin, 2000) — https://web.archive.org/web/20150906155800/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf
- The Single Responsibility Principle (Robert C. Martin, 2014) — https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html
- Solid Relevance (Robert C. Martin, 2020) — https://blog.cleancoder.com/uncle-bob/2020/10/18/Solid-Relevance.html
- CUPID: the back story (Dan North) — https://dannorth.net/blog/cupid-the-back-story/
- CUPID: for joyful coding (Dan North) — https://dannorth.net/blog/cupid-for-joyful-coding/
