# How the brain records things

Working document. Covers the second half of Q1 in `open-questions.md`: the shape of memory
events. Companion to `knowledge-model.md`, which covers what the brain is trying to know. This
one covers how evidence gets in.

Last updated: 2026 09 02

**Read this with D29 in mind.** The platform is multi user and every user has their own
isolated brain. The event kinds, the shape, the verbs and the consolidation rules are generic.
Where the text says "Adam" in a rule rather than an example, read "the owner".

---

## Six kinds of event

Distinguished by what the brain can learn from them, not by which feature emits them.

**Exposure.** The system put something in front of Adam. A news item ranked in, a session
generated, a topic suggested. Needed so the brain knows what he has already been told, and
so that ignoring something is distinguishable from never having seen it.

**Engagement.** What he did with it. Opened, completed, saved, abandoned partway, skipped.
Implicit preference signal, and the high volume layer.

**Judgment.** He evaluated something explicitly. Rated it, marked it too basic or too
advanced, accepted or rejected a proposal. Lower volume, much stronger signal.

**Assertion.** He told the system something directly. Profile input, a correction, "I did
this course three years ago." Lands in whichever store the claim belongs to, and authority
follows from that store rather than from the fact that Adam said it, see D23. The manual entry
path lands here.

**Decision.** A choice was made, with reasoning, in some context. A decision produces two
things and they behave differently: an immutable **record** that the decision was made,
which never decays, and a standing **claim** that the decision is in force, which can be
superseded or go stale. See "Decisions" below.

**Activity.** Work happened, on something, for some duration. Deliberately uninterpreted.
The stats substrate for behavioural patterns later, recorded now precisely because it
cannot be backfilled.

---

## The shape

One structure, five parts: a verb, a subject, a context, two timestamps, and a feature
specific payload.

```
verb:      exposed
subject:   topic: postgres-indexing, artifact: article-4471
context:   feature: news, run: r-8812
occurred:  2026-07-31T07:02   recorded: 2026-07-31T07:02
payload:   { rank: 3, score: 0.82, reason: "backend architecture interest" }
```

```
verb:      abandoned
subject:   topic: postgres-indexing, artifact: session-231
context:   feature: learning, run: r-9001
payload:   { sections_done: 2, sections_total: 6, stopped_at: "b-tree internals" }
```

```
verb:      asserted
subject:   topic: distributed-systems
context:   feature: core, source: manual-entry
occurred:  2023-09-01        recorded: 2026-07-31
payload:   { statement: "took a distributed systems course",
             claimed_level: "intermediate" }
```

```
verb:      decided
subject:   project: personal-platform
context:   feature: projects
payload:   { decision: "brain is its own service",
             rationale: "splitting memory across languages failed in v1",
             rejected: ["memory inside core backend"] }
```

Note the third example. Three years between the two timestamps, and recency weighting stays
correct. That is the whole reason both columns exist.

### Why this rather than a table per event type

Consolidation can reason over events it has never seen. It can ask "what topics has Adam
engaged with in the last thirty days, weighted by how strong each verb is as evidence", and
that query keeps working for a feature written years later. Per feature tables would mean
teaching the consolidator about every new feature.

The payload is opaque to generic consolidation and available to feature specific logic.
That is the extensibility seam, and the only place a feature gets to be special.

---

## Decided: one table for all events

**For.** Uniform consolidation queries. No schema change and no consolidator change when a
feature is added. Free cross feature ordering, so correlations across features are just a
time range query. Retention, scoring, idempotency and dedup implemented once.

**Against.** Payload is JSON, so no database level validation of feature specific fields.
Payload shape drift is invisible without a version field. No friction means everything gets
dumped in, so the filter test has to be enforced by discipline. Payload field queries are
slower than real columns.

**Why the against side is weaker than it looks:** volume per user. In a decade one user's
events might amount to a few hundred thousand rows, and every read and every consolidation
pass is scoped to one user by D29, so that is the number that matters rather than the size of
the table. Every scale objection is irrelevant at that volume, so ignore partitioning and hot
table concerns. The real costs are type safety and discipline.

Updated 2026 09 02: this previously read "single user", which stopped being true with D29. The
conclusion is unchanged, the reason is per user scope rather than a single account.

**Required from the start:**

- a payload version on every event, so shape drift is visible rather than silent
- an idempotency key, since consolidation is idempotent by decision and events may be
  re emitted on retry

**Decided: no side table for decisions.** See "Decisions" below for why they turned out not
to be a special case.

---

## Verbs

The brain owns the verb vocabulary. Features may add verbs.

**A new verb must declare its evidence strength and polarity.** Judged is stronger evidence
than engaged, which is stronger than exposed. Abandoned is negative polarity. A verb that
does not say where it sits cannot be weighed and is therefore unusable.

---

## Granularity is a per feature decision

Not a global rule. How finely a feature observes is decided when that feature is designed,
because what is worth recording depends on what the feature does and what we want the brain
to learn from it.

The brain owns the vocabulary so anything emitted can be weighed. The feature owns how
finely it observes.

The design question to answer per feature: what do I want the brain to learn from this, and
what is the cheapest signal that supports it? Each feature has a characteristic signal:

- **Learning sessions:** the quiz. Direct assessment beats any behavioural proxy, because
  it measures knowledge rather than guessing at it. This was already wanted for learning
  reasons and happens to solve the knowledge model's hardest problem.
- **News:** skips. Plentiful, and negative signal is strong.
- **Chat:** corrections. Adam correcting the system is the highest quality signal it will
  ever get.

Section level tracking within generated content is worth emitting where a feature already
knows its own structure. It is not worth building instrumentation for. Interaction level
tracking (scroll position, time on screen) is ruled out: it needs instrumentation
everywhere and time on screen is a bad proxy.

---

## Topic vocabulary

Mixed: seeded and grown. Three states.

**Canonical.** Seeded by Adam, or promoted after confirmation. Only canonical topics may
carry profile level claims.

**Provisional.** Auto created when a feature emits an event about a topic that does not
exist yet. Created silently, not queued for review, because putting every new topic in
front of Adam would make the review queue useless within a week. Usable as event subjects
immediately.

**Alias.** Merged into a canonical topic, with the old identifier still resolving.

**Merging** is the hard part and the reason the vocabulary needs managing at all. Without
it, "Postgres indexing", "database indexes" and "b-tree indexes" stay three unrelated
topics forever and the knowledge model fragments.

Approach, carried from the conclusion already reached on the wiki: similarity proposes
candidates, a model confirms and is allowed to say no, and merging happens during
consolidation rather than inline. Never similarity alone.

---

## Relations

Wanted, and expected to grow. **Relation types must be extensible**, same lesson as verbs:
a fixed enum will be wrong within a year.

Three kinds exist today and should stay distinct, since they have nothing to do with each
other beyond both being links:

- **Topic to topic.** Broader than, related to. Lets the knowledge model say Adam is solid
  on databases generally while actively learning one corner of it.
- **Entity to entity.** This platform project is that wiki project. The cross feature link
  table from `High-level/docs/platform-architecture.md` D12.
- **Memory to memory.** This evidence supports that claim. This claim contradicts that one.
  This supersedes that.

The third is what makes the brain inspectable. "Why do you believe this about me" is
answered entirely by evidence relations, and that is what makes it trustworthy in year
three rather than quietly ignored.

---

## Decisions

Decisions looked like they might need a first class side table, because they have a
lifecycle (active, then superseded) and an append only event log cannot express changing
status. They do not, and the reason is worth recording.

**A decision is not structurally special.** It is a thing that happened at a time, and a
thing with current status. That is exactly the shape of a semantic memory: derived from an
event, but carrying status and confidence that change over time. The architecture already
separates those, since memory items are distinct from memory events.

So: **the event records that a decision was made. A memory item holds its current state,**
with status and supersession, like every other memory. One event table, no side table.

### Decisions can go stale

Correcting an earlier claim in this document that records never decay. Two things are being
conflated:

- "On 2026 07 31 Adam decided to use hexagonal architecture" is a **record**. Permanently
  true, never decays, must never be downweighted into invisibility.
- "Adam uses hexagonal architecture" is a **claim** about the present. It can become false
  when he decides otherwise.

A decision produces both. The claim versus record split applies inside decisions rather
than making decisions purely records.

### Conflicting decisions before consolidation

Consolidation is what properly resolves a contradiction, by creating an explicit supersedes
relation. But retrieval has to behave sensibly in the window before that runs.

**Recency is a ranking bias, not a filter.** If two decisions conflict and consolidation has
not resolved it yet, retrieval returns both, ranks the newer first, and flags the conflict.
It does not silently drop the older one, because the reasoning behind the older decision is
frequently the thing that was actually needed.

Two refinements:

- Recency wins among decisions at the **same scope**. A recent narrow decision about one
  project should not override an older global one purely by being newer.
- This generalises past decisions to any conflicting claims. v1's memory context response
  already had a conflicts field, which is exactly what this is for.

**Scope comparison has three outcomes, not two.** Scope is a set of qualifiers rather than a
single value (D24), so scopes form a partial order rather than a line:

- one scope is strictly narrower, it wins regardless of age
- scopes are equal, recency wins
- scopes are incomparable, topic postgres versus entity operation rollout, and neither wins

The third case cannot be resolved by ranking and has to be surfaced through the conflicts
field. The natural implementation compares two scopes and returns a winner, which would be
silently wrong.

---

## Consolidation

Offline, periodic, and over the whole corpus **for one user**. The advantage is not just
batching: merging, pruning, and finding connections between things learned separately are
impossible while looking at one event at a time.

**The per user boundary is not a detail.** Under D29 users are fully isolated, so a
consolidation pass that reached across users would not merely produce a wrong claim, it would
move one person's evidence into another person's memory. Every pass takes a user and every
query inside it is scoped.

**Rules:**

- **Propose, never silently rewrite.** Human memory consolidation is lossy and
  reconstructive. A memory system that quietly rewrites itself into a plausible story is a
  failure mode, not a feature.
- **Consolidation never retires anything.** Nothing is deleted, and nothing is put away for
  being old or low scoring. Consolidation proposes, creates supersedes relations, and lets
  scores fall out of the evidence. Retiring is Adam's action alone. See D22.
- **A retirement or edit by Adam is recorded as a judgment event** against the original, so
  the next pass does not re propose what he just rejected. See D27.
- **Consolidation still processes directly written memory.** Writing immediately does not
  mean skipping consolidation. Anything written directly is still picked up on the next pass
  to be linked, merged, or checked for contradiction with existing memory. Immediate effect
  and later integration are separate concerns and both happen.

---

## Immediacy

Settled 2026 08 04. Q15 is closed.

Consolidation exists to infer claims from behaviour. When Adam records a fact there is
nothing to infer, so it does not wait.

- **Records write immediately.** A profile edit, "I took that course in 2023". Straight to
  its store, no queue, no wait.
- **Exposure, engagement and activity are evidence.** Generalizations from them wait for
  consolidation.
- **Judgment splits along the same line.** "This session was too basic" is a record about
  that session and lands immediately. "Adam prefers advanced material" is a claim about him
  and waits for repeated evidence.
- **Features write records directly. Features never write claims directly.** A claim about
  Adam still goes through evidence and consolidation.
- **An instruction mid conversation is a claim, not a record.** "Be more concise" applies to
  that conversation only, is recorded as an event, and never becomes a standing rule by
  itself. Standing rules are added to the rules list deliberately. See D25.

**Consequence that contradicts earlier advice:** the profile snapshot was going to be cached
for latency. A stale cache breaks immediacy. Cache with invalidation on write, which is
trivial for one user, but it has to be a rule rather than something discovered later.

**Correction 2026 08 04.** An earlier version had assertions writing immediately to
procedural memory, with "talk to me more concisely applies to the next message" as the
example. That single example generated the scope and confirmation problems tracked as Q15.
Both dissolved once it was recognised as a claim rather than a record.

---

## Still open in this area

- How the knowledge model represents level: discrete bands, a number, or something else (Q17)
- Whether "misconceptions corrected" is a real memory type or just a note

Closed since this was written: scope and confirmation on immediate writes (Q15, closed by D24
and D25), and whether decisions get a side table (Q16, resolved no, see "Decisions" above).

---

## Decisions recorded here

### D19. One table for all events, with an extensible schema


Every event, from every feature, in one table. Structure is a verb, a subject, a context,
two timestamps (when it happened and when it was recorded), and a feature specific payload.

Rationale: consolidation can then reason over events from features that did not exist when
it was written. Per feature tables would mean teaching the consolidator about every new
feature.

Scale objections do not apply, but the reason is not "single user", which stopped being true
with D29. It is that every read and every consolidation pass is scoped to one user, so the
volume that matters is the corpus per user rather than the size of the table. That stays
small no matter how many accounts exist.

Feature facing consequences:

- **Features may add verbs, but a new verb must declare its evidence strength and
  polarity.** A verb the brain cannot weigh is unusable.
- Every event carries a payload version and an idempotency key.
- Relation types are likewise extensible. A fixed enum will be wrong within a year.

Detail in `event-model.md`.
