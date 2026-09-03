# What a feature is, and what it may depend on

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

Isolation of implementation, coupling through contracts. This is what someone building a feature
needs to know. The brain's own internals are in `Brain/docs/`, and the platform wide thinking
these sit inside is in `High-level/docs/platform-architecture.md`.

---

## What a feature is

A feature is whatever registers itself with the core and talks to the brain through the two
contracts below. It owns its data and its frontend.

What sits inside it is its own business, whether that is one thing or several smaller ones
bundled under a general name. Either shape looks identical from the core's and the brain's side,
so there is nothing to standardise and nothing to declare.

## Features own their data

A wiki concept node belongs to the wiki. A learning session belongs to learning. The brain never
holds a copy of any of it.

**A feature sends the brain something once, and that is the whole exchange.** What it sends is
what the brain should learn from that feature, which by the filter in
`Brain/docs/knowledge-model.md` means the things that change an output outside that feature. It
is always a small, deliberately chosen set rather than a feed of everything that happened.

**The evidence comes with it, as readable text that stands on its own.** "Worked through B tree
internals, got the page split case wrong twice, said the material was too basic" is something you
can read and judge years later without asking anything else. Every event carries one, in a field
of its own, see the event shape in `Brain/docs/event-model.md`. The brain never holds a pointer
into a feature and follows it later to find out what it meant.

That matters more than it looks. The trust model rests on "why do you believe this about me"
always being answerable. A pointer makes that conditional on the feature still being up and the
row still existing, so deleting a session would quietly leave the brain holding a claim it can no
longer justify. Self contained evidence makes the answer unconditional, and it is also what keeps
the promise that nothing in the brain is ever destroyed, since a feature cannot destroy it by
deleting its own data.

The cost, worth being clear about: write time is the only chance. Whatever a feature leaves out
of that payload is gone, because an event records a moment and cannot be backfilled. Payloads
should err generous.

The brain embeds what it holds, so memory becomes semantically retrievable. It does not embed
feature content, because it does not have any.

The brain owns what that data means about the person, and that is the whole division.

## Features do not call each other

All cross feature access goes through the brain. This is written as a rule rather than a
preference, and here is what goes wrong without it: with N features calling each other directly
you get N squared coupling, every feature has to know the shape of every other one, and a
refactor anywhere breaks something somewhere else. At the size this platform is aiming for that
is the difference between adding a feature being cheap and adding a feature being a project.
It is the rule to defend hardest.

Note that a browser calling several features to draw one page is not a feature calling another.
That is the user, holding their own token.

## The brain does not connect to feature databases

A feature exposes a narrow read API over its own domain, and the brain calls it. The feature
keeps fast local access to its own database, and cross feature access is one extra hop through
the brain, which is milliseconds and fine.

The alternative, the brain holding direct database connections to every feature, was considered
and dropped. It couples the brain to every feature's internal schema, breaks when a feature
refactors, bypasses the feature's own rules so the brain reimplements them badly, and spreads
credentials around.

## The brain stays out of a feature's domain

It routes, merges, ranks, and remembers. It does not decide what makes a good wiki node, or when
a project task is blocked. Ranking and merging are its own domain, so those are not violations.

Nothing structural enforces this. It is judgment, applied each time a call gets added, and the
place it will go wrong if unwatched is a context call that starts returning advice instead of
facts.

## Brain access sits behind one port per feature

One adapter, not scattered HTTP calls through the codebase.

Worth being clear about why, because the obvious reason is the wrong one. This is not for
portability or future extraction. It is for testability, so a feature can run against a fake
brain in its own test suite, and so the implementation can be swapped without touching callers.

## How finely a feature observes is up to it

Not a global rule. It depends on what the feature does and what the brain should learn from it,
so it gets decided when that feature is designed. The brain owns the verb vocabulary, so
anything emitted can be weighed, and the feature owns how closely it looks.

One thing ruled out everywhere: interaction level tracking, scroll position and time on screen.
It needs instrumentation in every corner and time on screen is a bad proxy for anything.

## Records apply immediately, claims wait

Consolidation exists to infer claims from behaviour. When the owner records a fact there is
nothing to infer, so it does not wait.

- **Records write immediately.** A profile edit, "I took that course in 2023", "this session was
  too basic". Each states a fact, so there is nothing to get wrong.
- **Exposure, engagement and activity are evidence.** Generalisations from them wait.
- **Features write records directly, and never write claims directly.** A claim about the person
  goes through evidence and consolidation.

Writing immediately does not mean skipping consolidation. Anything written directly is still
picked up on the next pass to be linked, merged, or checked for contradiction. Immediate effect
and later integration are separate things and both happen.

One thing worth not getting wrong here: "talk to me more concisely", said mid conversation, is
not a record. It is a claim about the person wearing an imperative, and letting one sentence
create a standing global rule is generalising from a single data point, which the rest of the
design refuses to do everywhere else. `Brain/docs/trust-model.md` covers where those go instead.

## Reminders are delivery, not a feature

A feature sends a request: tell the user this, at this time or under this condition. Nothing
gets linked. Self authored reminders, "call the dentist", "look at the Q3 numbers on Friday", go
through the same service with the user as the source instead of a feature.

**A reminder is never read as a signal about the person.** It is not evidence, it is not a
preference, and it never influences ranking, generation or any other output. It is a note the
user wants delivered back to himself at a time.

Two things follow. The notification service does not write to the brain, since a reminder
changes no output and so it is a note rather than memory, and delivery flows one way. And
reminders are not standing rules: "use TDD" and "plan before prompting" are rules about how the
system should behave and belong to procedural memory, while a reminder is delivered and then
done.

`High-level/docs/platform-architecture.md` covers where the delivery service lives.

---

## The two contracts

Both stay small. If either drifts large, something has gone wrong.

**Downward, what the brain offers a feature**

- a named context call per feature, shaped when that feature is built
- get profile snapshot
- search, meaning search memory, not search everything the platform knows
- record event
- propose memory candidate

The first entry used to read "get context for a task", which was close enough to v1's rejected
single magic call to be the same mistake under a new name. `Brain/docs/brain-architecture.md`
has how per feature calls work instead.

**Upward, what a feature offers the brain**

- a narrow read API over its own domain, shaped per feature
- the events it emits

Only the brain calls a feature's read API. That is a topology rule rather than a security one:
it is what keeps features from linking to each other directly.

**Both directions carry a verified user identity, taken from the token rather than from a
parameter.** This one is firm, and what goes wrong without it is not a wrong answer but one
person's data reaching another account. There is no unscoped call in either contract.

---

## What we are not doing

**A generic search envelope every feature conforms to.** Designed to avoid touching the brain
when adding a feature. At five features that cost is not real, and flattening every domain into
one shape loses structure that matters. Speculative abstraction.

**The brain connecting directly to feature databases.** Covered above.
