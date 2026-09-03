# What a feature is, and what it may depend on

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

The rules a feature author has to know. Isolation of implementation, coupling through
contracts, and nothing else.

The brain's own internals are not here, see `Brain/docs/`. The platform wide decisions these
rules sit inside are in `High-level/docs/platform-architecture.md`.

---

## Decisions

### D2. Features own their data. The brain owns what that data means about you


A wiki concept node belongs to the wiki. A learning session belongs to learning. What
crosses into the brain is a reference, a summary, an embedding, and the events. Never the
artifact itself.

### D3. Features never call each other


All cross feature access goes through the brain. This is the rule that stops N features
becoming N squared coupling, and it is the one to defend hardest.

### D4. The brain does not connect to feature databases. Features expose read APIs


Considered and rejected: the brain holding direct database connections to every feature.
Reasons it was rejected are in "Considered and dropped" below.

Instead: a feature exposes a narrow read API over its own domain, and the brain calls it.
The feature keeps fast local access to its own database. Cross feature access is one extra
hop through the brain, which is milliseconds and fine.

### D7. The brain never makes a decision that belongs to a feature's domain


It routes, merges, ranks, and remembers. It does not decide what makes a good wiki node or
when a project task is blocked. Ranking and merging are its own domain, so those are not
violations.

### D8. Each feature wraps brain access behind one internal port


Not scattered HTTP calls through the codebase. One adapter.

Note the justification changed during discussion. This is **not** for portability or future
extraction, see D9. It is for testability, so a feature can run against a fake brain in its
own test suite, and so the implementation can be swapped without touching callers.

### D10. Feature tiers


**Dissolved 2026 09 03 rather than answered.** This defined three tiers, federated, embedded
and linked. Two of them were empty and the third was every feature, so the model sorted
nothing.

Embedded meant a feature living inside the brain's process and database with its UI drawn by
the core shell. Its only occupant was ever reminders, which D13 removed by ruling it is not a
feature at all. D14 then removed what fed it: embedded requires the brain's own language, and
once each feature picks its language at its own planning phase, landing in Python is a
coincidence rather than a reason to move a feature inside the brain. A written route into the
brain is also the exact pressure D1 was raised against, since a general service absorbs what
sits next to it because it is always the path of least resistance. Linked was never occupied
either, and describes a link in the navigation rather than a feature.

**What replaces it, and it is deliberately loose.** A feature is whatever registers itself with
the core and talks to the brain through the two contracts below. It owns its data and its
frontend. What sits inside it is its own business, whether that is one thing or several smaller
ones bundled under a general name, and either shape looks identical to the core and the brain.

Nothing else needed to be a rule. Everything the table actually enforced is already stated
elsewhere: features own their data (D2), the brain does not reach into their databases (D4),
and access runs through one port (D8). What is left is a judgment taken per feature when that
feature is planned, which is where it belongs, and the tiers were taking it years early.

The one thing the table did describe that had nowhere else to go, the memory centre and the
review queue, turned out not to be features either. They are the brain's own surface over its
own data, see D32.

### D13. Reminders and notifications are a delivery capability, not a feature


A feature sends a request: tell the user this, at this time or under this condition. Nothing
gets linked. Self authored reminders ("call the dentist", "look at the Q3 numbers on Friday")
go through the same service with the user as the source instead of a feature.

This corrected an earlier misclassification of reminders as an embedded feature, a tier that
has since dissolved, see D10.

**Clarified 2026 09 02, and this is the part that was ambiguous.** A reminder is a note the
user wants delivered back to himself at a time. **The system never reads a reminder as a
signal about him.** It is not evidence, it is not a preference, and it never influences
ranking, generation or any other output.

Two consequences follow, and they are worth stating rather than inferring:

- **The notification service does not write to the brain.** By the filter in
  `Brain/docs/knowledge-model.md`, a reminder changes no output, so it is a note rather than
  memory. Delivery flows one way: features and the user request it, nothing comes back.
- **Reminders are not procedural rules.** An earlier version of this decision used "use TDD"
  and "plan before prompting" as examples, which are standing rules about how the system
  should behave, and they belong to D25 and procedural memory. The examples were the
  ambiguity, not the design. A rule shapes behaviour, a reminder is delivered and then done.

See D30 for where this capability lives.

### D20. Observation granularity is a per feature decision


Not a global rule. How finely a feature observes is decided when that feature is designed,
since it depends on what the feature does and what the brain should learn from it. The
brain owns the verb vocabulary so anything emitted can be weighed; the feature owns how
finely it looks.

Ruled out globally: interaction level tracking (scroll position, time on screen). Needs
instrumentation everywhere and time on screen is a bad proxy.

### D21. Records apply immediately. Claims wait for consolidation


Consolidation exists to infer claims from behaviour. When the owner records a fact there is
nothing to infer, so it does not wait.

- **Records write immediately.** A profile edit, "I took that course in 2023", "this session
  was too basic". Each states a fact, so there is nothing to get wrong.
- **Exposure, engagement and activity** are evidence. Generalizations from them wait.
- **Features write records directly. Features never write claims directly.** A claim about
  the user goes through evidence and consolidation.

**Writing immediately does not mean skipping consolidation.** Anything written directly is
still picked up on the next pass to be linked, merged, or checked for contradiction.
Immediate effect and later integration are separate concerns and both happen.

**Amended 2026 08 04.** This was titled "assertions apply immediately" and used "talk to me
more concisely affects the next message" as its example. The example was wrong, and it was
the source of a great deal of accidental complexity. "Adam wants concise prose" is a claim
about Adam, not a record, and letting one sentence create a standing global rule is
generalising from a single data point, which this document refuses to do everywhere else. An
instruction mid conversation is a claim wearing an imperative. See D25.

---

## The two contracts

Both stay small. If either drifts large, something has gone wrong.

**Downward, what the brain offers a feature**

- a named context call per feature, shaped when that feature is built, see D28
- get profile snapshot
- search
- record event
- propose memory candidate
- index document

The first entry read "get context for a task" until 2026 08 04. That wording was close enough
to v1's rejected single magic call to be the same mistake under a new name. Replaced by D28.

**Upward, what a feature offers the brain**

- a narrow read API over its own domain, shaped per feature
- the events it emits

Only the brain calls a feature's read API. That is a topology rule rather than a security
one: it is what keeps features from linking to each other directly.

**Both directions carry a verified user identity**, taken from the token rather than from a
parameter. Under D29 there is no unscoped call in either contract.

---

## Considered and dropped

**The brain connecting directly to feature databases.** Couples the brain to every
feature's internal schema, breaks when a feature refactors, bypasses the feature's own
rules so the brain reimplements them badly, and spreads credentials. Replaced by D4.

**A generic search envelope every feature conforms to.** Designed to avoid touching the
brain when adding a feature. At five features that cost is not real, and flattening every
domain into one shape loses structure that matters. Speculative abstraction, dropped.
