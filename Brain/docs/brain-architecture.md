# The brain

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

How the brain is built. Everything here is internal: you need it to write the brain and you do
not need it to understand the platform.

What the rest of the system needs to know about the brain is elsewhere, and is not repeated
here: D1, D11, D15, D33 and the memory layers are in
`High-level/docs/platform-architecture.md`, and the two contracts are in
`High-level/docs/feature-contract.md`.

**Other documents in this folder**

- `trust-model.md` — authority, application, and how belief is corrected
- `knowledge-model.md` — what the brain is trying to know about a user
- `event-model.md` — how evidence gets recorded, and the event shape

---

## Decisions

### D5. Per feature code in the brain, written as needed


There is no generic envelope every feature must conform to. When feature B needs feature
A's data, we add the endpoint to A and the reader to the brain at that moment, and decide
the right shape for that case.

Containment rule: that per feature code lives in one thin module per feature inside the
brain, modules do not reach into each other, and they only translate and route. The point
is that the brain stays legible at feature eight rather than becoming the biggest and most
tangled thing in the system.

### D6. Unified interface facing out, separate modules facing in


A feature asks the brain a question and gets an answer. Whether that answer came from the
brain's own memory, from another feature, or from both merged, is not the feature's
business. The moment a feature has to know where data lives, the topology has leaked into
every caller.

Inside the brain, memory logic and broker logic are separate modules. One service for now.
Split only if there is ever a real reason.

### D16. Retrieval evaluation exists from early on


A frozen set of example situations, a seeded memory state, and what should and should not
come back. A runner scores recall, precision, and ranking. Run on every change.

Why: the brain's core job is a judgment with no obviously correct answer, so it cannot be
unit tested normally. Without this, "is the brain getting better" is unanswerable, and the
goal is a system that improves over years.

Sequencing: not first, since there must be a schema and a crude retrieval path to run
against. Immediately after the simplest retrieval works, so everything from that point on
is measured.

Free eval data: every accept or reject in the review queue is a labelled example. Keep
rejections rather than discarding them. Impossible to reconstruct later.

Consolidation evaluation is a separate and harder problem (given these events, did it learn
the right thing) and comes later.

### D17. Consolidation is idempotent and re runnable from the start


Same window processed twice produces no duplicate proposals. Good practice regardless, and
it happens to be the only thing that makes adding durable orchestration later a config
change rather than a rewrite.

There is no durable orchestration today and nothing assumes one. If the consolidation schedule
ever needs it, Temporal is the candidate, and some features may end up running their own jobs
instead. Because of this decision that stays a config change whenever it is made.

### D28. Retrieval is per feature calls over shared primitives


**What v1 actually got wrong was genericity, not specificity.** One call had to serve every
caller, which is exactly why it needed a provider interface plus a switch on workflow type
inside. The switch was a symptom. Name the call "context for a learning session" and it
disappears, because the routing already happened at the function name.

Same for the response being nine kinds of thing. A learning session genuinely needs many kinds
at once. The problem was that every caller got the same nine whether they made sense or not.
Richness was never the issue, uniformity was.

**Structure, and it is D5 applied to retrieval:**

- **Facing out**, named per feature calls, living in that feature's thin module in the brain,
  written when that feature is built rather than guessed at now.
- **Facing in**, a small set of shared primitives: profile lookup, knowledge for topics,
  history for a subject, scoring, conflict detection. Features never see these, so D6 holds and
  a feature never merges anything itself. They exist once, so ranking logic is not
  reimplemented in five modules.

**The D7 line, and it is the thing that will actually go wrong if unwatched.** The brain
returns what it knows about the user bearing on the task. The feature decides what to do about it.
So the brain says competent at this topic, last evidenced in March, two misconceptions
corrected, adjacent topics he is solid on, one conflicting claim. Learning reads that and
decides what to teach. The brain never returns "teach B tree internals next", because that is
pedagogy and pedagogy is learning's domain. Nothing structural prevents this, only judgment
each time a call is added.

**Response shape:**

- Calls are bespoke, items are not. Every returned item carries id, content, store, provenance,
  confidence, currency, scope, evidence references, status.
- **Evidence references are included by default, not on request.** That is what makes "why do
  you believe this" answerable where it is used rather than only in the memory centre, and it
  is what decides whether the thing is still trusted in year three.
- A call declares whether it is asking about now or about history, since that changes how
  currency is weighted, see D26.
- Conflicts come back as conflicts rather than being silently resolved, see D24 and D26.

**Every call and every primitive takes a user scope, and it is not a parameter.** The scope
comes from the verified identity in the token, never from an argument the caller supplies.
That is a security property rather than an implementation note, which is why it lives in the
decision: a call that accepts a user id as data is one bug away from returning another
person's memory. See D29.

**This is not the generic envelope that was rejected.** That one was an envelope every
_feature_ conforms to when exposing its own data, which is the upward contract. A consistent
item shape on the way down is a different thing and costs nothing.

**Milestone consequence.** D18 builds no features, so there are no named calls to write and the
context packet has no caller. Milestone one builds the shared primitives, and the evaluation
harness is the first caller. Per feature modules arrive with the first feature, shaped by what
it really needs. That is better than designing the learning context call before learning
exists.

### D32. The brain owns its own surface


**Decided 2026 09 03.** The memory centre is the brain's own UI over its own data. It is not a
feature and does not use the feature contract. It holds the profile editor, the general item
browser D27 turns it into, the timeline, the standing rules list from D25, and the review queue
described in `trust-model.md`.

**Why this needed saying at all.** D10's tier table was the only place these were described,
as "embedded" things that stored in the brain and had their UI drawn by the core shell. D10
dissolved on 2026 09 03, which left them with no home and made "is the memory centre a feature"
answerable either way.

**Why they are not features.** A feature owns domain data and exposes a narrow read API that
the brain calls, see D2 and D4. These own no domain data at all. They read and write the
brain's own stores, and there is nothing for the brain to call, so putting them behind the
feature contract would mean the brain calling itself over HTTP to reach its own tables.

**What follows:**

- No feature registry entry, no read API, no thin module in the brain. None of the machinery
  applies, because none of it has anything to do.
- It still has a real frontend of its own, TypeScript per D14, talking to the brain. It is not
  server rendered by the shell, which is what the dissolved tier assumed.
- It sits behind the same authentication the core issues and carries the same verified user
  scope as any other caller, see D29 and D31. Outside the feature contract is not outside the
  platform.
- Edits made here still record judgment events, see D27. That is the brain writing its own
  log, not a feature using the upward contract.

Not first milestone work, see the note under D18, but a meaningful share of the real build.

---

## Considered and dropped

**One magic GetMemoryContext call.** v1 had a single method taking an arbitrary task string
and returning nine kinds of thing. It did not fit, and ended up needing a provider interface
plus a switch on workflow type. Let callers ask for what they need.

**A weighted relevance formula with fixed constants.** v1 had authority times 0.35 plus
similarity times 0.25 and so on. There is no ground truth to tune against and no way to
tell if a change helped. Start with filtering and simple ordering, earn the weights through
evaluation.

**A side table for decisions.** They have a lifecycle (active, then superseded) that an
append only event log cannot carry, which looked like it needed special handling. It does
not: a decision is a thing that happened plus a thing with current status, which is exactly
the shape of a semantic memory, and memory items are already separate from memory events. It
also corrected an error, that decisions were pure records that never decay: a decision produces
a permanent record *and* a standing claim that can be superseded. Detail in `event-model.md`.

**Section level instrumentation as a global requirement.** Considered as the passive signal
that would maintain the knowledge model. Superseded by direct assessment: the quiz Adam
already wanted at the end of learning sessions measures knowledge rather than guessing at
it. Emit section detail where a feature knows its own structure for free; do not build
tracking for it. See D20.

---

## Carried from v1

Ideas worth keeping, as ideas, not code. v2 is a fresh build.

- **Events then consolidate.** Features record what happened; a background pass interprets
  it later. Refined in D21: features never write _claims_ directly, but records are fine,
  and explicit assertions do not wait for consolidation to take effect.
- **The review queue.** Proposals with evidence and confidence that the owner accepts or rejects.
  This is the trust mechanism and the reason the system stays inspectable.
- **Authority ordering.** Explicit user input outranks confirmed suggestion outranks
  repeated evidence outranks inference. Kept, but demoted by D23: it applies within a store
  rather than across all of memory. The false precision of fixed decimal weights is dropped.
- **The memory centre UI concept.** A place to see and correct what the system believes.
  Carried, and now a decision of its own, see D32.
- **Not markdown as the storage substrate.** Markdown is a fine authoring format for things
  written by hand. It is a bad query substrate. Storage and retrieval is Postgres plus
  pgvector.

One thing v1 wanted and could not do: letting an agent find relevant data itself rather than
being fed it. v2's shape, a brain sitting in front of every feature's read API, is the shape
that makes that possible. Not designed for, but worth noticing.
