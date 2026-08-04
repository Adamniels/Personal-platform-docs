# How the brain records things

Working document. Covers the second half of Q1 in `open-questions.md`: the shape of memory
events. Companion to `brain-knowledge-model.md`, which covers what the brain is trying to
know. This one covers how evidence gets in.

Last updated: 2026 08 01

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
this course three years ago." Highest authority. The manual entry path lands here.

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
time range query. Retention, decay, archival, idempotency and dedup implemented once.

**Against.** Payload is JSON, so no database level validation of feature specific fields.
Payload shape drift is invisible without a version field. No friction means everything gets
dumped in, so the filter test has to be enforced by discipline. Payload field queries are
slower than real columns.

**Why the against side is weaker than it looks:** single user. In a decade this table might
hold a few hundred thousand rows. Every scale objection is irrelevant at that volume, so
ignore partitioning and hot table concerns. The real costs are type safety and discipline.

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
  table from `architecture-decisions.md` D12.
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

---

## Consolidation

Offline, periodic, whole corpus. The advantage is not just batching: merging, pruning, and
finding connections between things learned separately are impossible while looking at one
event at a time.

**Rules:**

- **Propose, never silently rewrite.** Human memory consolidation is lossy and
  reconstructive. A memory system that quietly rewrites itself into a plausible story is a
  failure mode, not a feature.
- **Archive, never delete.** Archived items go somewhere readable, so Adam can see what was
  archived and why. Nothing is irreversible, and a restore feature can be added later
  without needing the data to have been kept deliberately.
- **Consolidation still processes directly written memory.** Writing immediately does not
  mean skipping consolidation. Anything written directly is still picked up on the next pass
  to be linked, merged, or checked for contradiction with existing memory. Immediate effect
  and later integration are separate concerns and both happen.

---

## Immediacy

Partly settled, partly open. The open part is tracked as Q15 in `open-questions.md`.

**Settled:** consolidation exists to infer claims from behaviour. When Adam states something
directly there is nothing to infer, so it does not wait.

- **Assertions write immediately**, at highest authority, straight to profile or procedural
  memory. "Talk to me more concisely" applies to the next message.
- **Exposure, engagement and activity are evidence.** Generalizations from them wait for
  consolidation.
- **Judgment splits along the same line.** "This session was too basic" is an assertion
  about that session and is recorded as fact immediately. "Adam prefers advanced material"
  is a claim about him and waits for repeated evidence.
- **Features write records directly. Features never write claims directly.** This restates
  v1's rule correctly: a decision is a record of something that happened, so there is
  nothing to get wrong. A claim about Adam still goes through evidence and consolidation.

**Consequence that contradicts earlier advice:** the profile snapshot was going to be cached
for latency. A stale cache breaks immediacy. Cache with invalidation on write, which is
trivial for one user, but it has to be a rule rather than something discovered later.

**Still open, see Q15:** scope. If Adam says "be more concise" during a learning session,
does that apply to learning or everything. Also the confirmation experience: a popup
answered inline versus something landing in the review queue, and which kinds of learning
warrant which. Likely decided per feature, when designing what each feature can learn from
an interaction.

---

## Still open in this area

- Scope and confirmation on immediate writes (Q15)
- Whether decisions get a side table
- How the knowledge model represents level: discrete bands, a number, or something else
- Whether "misconceptions corrected" is a real memory type or just a note
