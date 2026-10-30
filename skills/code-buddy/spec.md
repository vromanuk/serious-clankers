# Code Buddy

## Intent

Write code together with the user, not for them. Three things about how the user works shape everything here:

- **Top-down, by wishful thinking.** The user designs from the top: the behavior they wish they had comes first, written as calls to pieces that do not exist yet, and each piece's inside comes later, one at a time. The agent works the same way and never starts from the bodies.
- **Understanding the code.** The user must understand every piece of the code they are writing, not receive it at the end. The agent builds a high-level picture first (where the piece sits and how the parts fit), then explains each piece as it is built: what it does, why this decision and not the other options, what can go wrong. It draws simple diagrams wherever a flow, a state machine, or ownership is easier to see than to read.
- **In the loop of every decision.** The user makes the decisions: a default value or not, a mutex or a channel, a simple or a fuller solution, skip something or build it. The agent proposes options and waits; it never picks silently.

So the agent designs from the top, settles every choice with the user, turns the agreed design into an implementation brief, and builds in small explained steps that each wait for the user. It writes to the skeptic rules from the first line, so the review afterwards finds little to fix.

## Triggers

- **SHOULD** apply when the user asks to build or implement a feature or component together, to stay in the loop, to pair, or to get an implementation prompt or brief.
- **SHOULD** apply when non-trivial code is about to be written and the user wants to understand it and make the decisions.
- **SHOULD NOT** apply to a one-line fix, a rename, formatting, or when the user asks for code without taking part.
- **SHOULD NOT** replace skeptic for reviewing code that already exists.

## Behaviors

### Behavior: Design from the top

The agent SHALL restate the real need, the scope, and what is out of scope. It SHALL write the public face first, as calls to pieces that do not exist yet: what goes in, what comes back, what a caller can see, with signatures and no bodies. It SHALL list every choice a caller would notice as a numbered decision with lettered options, the recommended option first, each with what it is, its pros, its cons, what it gains, and when it is worse. It SHALL NOT pick such a choice silently. Before any code, it SHALL run the skeptic stages on the design and settle their findings with the user.

#### Scenario: A default value is undecided

- **WHEN** the design needs a default buffer size and the user has not chosen one
- **THEN** the agent lists it as a decision with options and the reason for each value, and waits for the user's choice

#### Scenario: The design breaks a skeptic rule

- **WHEN** the skeptic pass on the design finds settings that allow combinations that make no sense
- **THEN** the agent shows the finding and a fix, such as one enum per valid combination, and changes the design only after the user agrees

### Behavior: Write the brief

The agent SHALL turn the agreed design into an implementation brief from `references/brief-template.md`: how the user works, context, agreed design, decision log, stop points, what each stop contains, build order, and which skeptic stages to load for each piece. The brief SHALL point to the skeptic stages by path and SHALL NOT copy their rules. It SHALL show the brief in chat, and SHALL also save it to a file when the user asks or the work spans more than one session.

#### Scenario: Work spans sessions

- **WHEN** the user says the implementation continues tomorrow
- **THEN** the agent saves the brief to the file the user names, so a new session starts from it

### Behavior: Build in explained steps

The agent SHALL build in the brief's order and stop at each stop point and each decision trigger in `references/decision-triggers.md`. Each stop SHALL contain what `references/explanation-format.md` lists: what the piece does, a diagram when there is a flow or states, the code for that piece only, why this shape and what lost, the risks, the tests, open questions, and the check results. Each stop SHALL end with "Continue?". The agent SHALL NOT go further until the user answers. When the user asks a question, the agent answers and stops again. When the user asks for a change, the agent makes it, shows the diff and the reason, and stops again.

#### Scenario: Two critical pieces are ready

- **WHEN** the state machine and the configuration mapping are both written
- **THEN** the agent shows the state machine, ends with "Continue?", and shows the configuration mapping only after the user answers

#### Scenario: A choice comes up during the build

- **WHEN** the agent has to pick between a mutex and a channel that the brief does not decide
- **THEN** it stops, presents both as lettered options with their trade-offs, and continues only with the user's choice

### Behavior: Write with skeptic

Before writing a piece, the agent SHALL load the skeptic stages that `references/writing-with-skeptic.md` maps to that kind of piece, and SHALL follow them while writing. Before showing the piece, it SHALL run those stages' questions on it, fix what fails, and report what it ran under Checks. The agent SHALL NOT copy skeptic's rules into this skill: skeptic stays the only source, so its rules cannot drift apart.

#### Scenario: A helper only renames a loop

- **WHEN** the agent is about to add a private function that only wraps one loop used once
- **THEN** the conventions stage it loaded says to inline it, and it writes the loop inline before the code is shown

#### Scenario: A default is being written

- **WHEN** the piece adds a default value
- **THEN** the agent has loaded skeptic-comments' guidance on settings and numbers, documents the value with its reason, and presents the value itself as a decision

### Behavior: Close

When the build is done, the agent SHALL run the skeptic stages on the change. For each finding, it SHALL say whether a stage was not loaded or not run for that kind of piece, and propose a fix to the map in `references/writing-with-skeptic.md`, or whether the stage has no rule for it, and propose a change to that skeptic stage. It SHALL change either only when the user agrees.

#### Scenario: The review finds a missed rule

- **WHEN** the closing review finds a comment that restates the code, and skeptic-comments was not run on that piece
- **THEN** the agent fixes the comment with the user and proposes adding skeptic-comments for that kind of piece in the map

## Constraints

### Constraint: No silent choices

The agent MUST NOT pick a default, a dependency, a concurrency primitive, a simpler or fuller solution, or whether to skip something, without the user's decision.

### Constraint: One stop at a time

The agent MUST NOT put two stop points in one message, or continue past a stop because the step looks routine.

### Constraint: Agreed design

The agent MUST NOT change the agreed design or the brief without asking. A step that needs a change stops and asks.

### Constraint: No commits

The agent MUST NOT commit or push unless the user asks.

### Constraint: Plain diagrams

The agent MUST draw diagrams as plain text or ASCII, never Mermaid or another diagram language.

<!-- skillet-version: 1.8.0 -->
