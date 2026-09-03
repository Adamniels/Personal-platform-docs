# What the brain should know

Working document. Covers what the brain is trying to know about a user. Deriving concrete event
shapes from this is the companion job, and it is in `event-model.md`.

Last updated: 2026 09 03

**Read this as multi user.** The platform is multi user and every user has their own
isolated brain. The six categories, the filter and the claim versus record distinction are
generic and apply to any user. The specific claims, seeds and examples throughout are Adam's
own case, since he is the first user and the one seeding it. Where this document says "Adam"
in a rule rather than an example, read "the owner".

---

## Method

Rather than asking "what should features record", we worked backwards: what do we want the
brain to know that nothing today could tell us, then what evidence would justify it, then
what event carries that evidence. The taxonomy comes out as a result instead of a guess.

Same list doubles as the first draft of retrieval evaluation cases, since a claim you want
the brain to hold is a thing you want retrieval to surface.

**This is not blocked on the feature list.** There is a core set of things the brain should
know that is feature independent, plus a lot that Adam fills in by hand. Each feature then
decides what it contributes when it is built.

Design consequence, and this is load bearing: **the event schema must be extensible by
construction.** A type plus a structured payload plus provenance, never a closed list of
every event that will ever exist. We are designing a spine and a shape, not a complete
taxonomy.

---

## The filter

The single test for whether something belongs in the brain at all:

> **A fact belongs in the brain only if you can name the output that changes because of it.**

If you cannot name one, it is a note, not a memory. Notes are fine, they just live in a
feature.

Worked examples. "Adam prefers depth over overviews" passes: it changes session generation
and news ranking. "Adam opened the app at 11pm" fails: nothing changes.

Combined with feature ownership, this generalises into the promotion rule:

> **A feature promotes something to the brain when it changes an output outside that
> feature. Otherwise it keeps it.**

So a decision about Operation Rollout's item cap stays in Operation Rollout. A decision
about how Adam wants architecture explained goes to the brain, because it changes
everything.

---

## Six categories

| Category | Source | Lifetime | Priority |
|---|---|---|---|
| Stable identity | manual | years | high, and nearly free |
| Current state | manual and inferred | months | high |
| Knowledge model | seeded then inferred | continuous | highest |
| How to work with Adam | manual, refined by feedback | slow | high value per effort |
| Behavioural patterns | derived | continuous | deferred, see below |
| History and continuity | promoted by features | permanent | high, underrated |

### Stable identity

Changes yearly or never. Entered by hand, high authority, anchors everything else.
Career direction and stage, long term technical interests, engineering values, what he
wants to be known for.

### Current state

Changes monthly. What he is actively working on, real time allocation, current focus,
what is blocking him, deadlines coming up. This is what makes anything feel current rather
than generic.

### Knowledge model

What he knows and how well. Topics he is solid on, actively learning, touched shallowly,
deliberately not pursuing. Also misconceptions that have been corrected, which is underused
elsewhere. Whether a corrected misconception is a memory type of its own or simply a note is
settled when the schema is written.

Highest priority because one investment improves several outputs at once (learning sessions
stop repeating known material, news ranking stops surfacing basics, explanations start at
the right level) and because it cannot be maintained by hand. If the brain does not do it,
nobody does.

### How Adam wants to be worked with

Procedural memory. Mental model first then structure then code. Pair every recommendation
with the question rather than asking open ended. Push back rather than agree. No
speculative abstraction without current pressure. Concise prose, no hyphens or en dashes
or underscores.

Cheap to capture, compounds into every single interaction. High value per unit of effort
even though it is not exciting.

### Behavioural patterns

**Interpretation deferred. Recording is not.**

The expensive and risky part is the interpretation, not the recording. "You tend to
procrastinate on X" is an interesting fact that does not obviously change any output, and
it runs into the creepiness problem flagged in v1.

But the recording is cheap, factual, and the one thing that cannot be done retroactively.
So: **do not build behavioural inference, but make sure events carry enough structure that
the stats are derivable later.** If in two years the question is how long Adam stays on one
project before switching, that has to have been recorded all along.

Same events then consolidate split, applied to this category. The habit tracker idea lives
here and is a later feature.

### History and continuity

What was learned, when, at what depth. Decisions made and why. Things tried that did not
work. What Adam has already been told.

Underrated. Four projects running across years means "why did I decide this" gets asked
constantly, and today the answer is scattered across Notion pages and chat logs. The wiki
solves this for code. Nothing solves it for the decisions themselves.

---

## Claim versus record

Load bearing, and it belongs in the schema rather than being handled by decay rates.

- A **claim** asserts something about now. "Adam is focused on embedded systems." Its
  currency falls as time passes without supporting evidence.
- A **record** states that something happened. "Adam decided the brain is its own service,
  because splitting memory across languages failed in v1." It is exactly as true in 2030 as
  today. Currency is undefined for records rather than merely slow, so staying visible is a
  consequence of what a record is rather than a rule anyone has to enforce. See how currency
  works in `trust-model.md`.

This also resolves where current state ends and history begins. The boundary is not time,
it is whether the fact asserts something about now or records something that happened.

**One thing can produce both.** A decision generates an immutable record that it was made,
plus a standing claim that it is in force. The record never decays; the claim can be
superseded. So the split applies within a single event rather than sorting events into two
buckets. See "Decisions" in `event-model.md`.

---

## Timestamps

**Events need when it happened separate from when it was recorded.**

Adam wants to backfill old knowledge ("I did this course three years ago"). Without two
timestamps, everything backfilled looks like a fresh signal and recency weighting breaks
the moment the feature is used. One extra column now, versus rewriting every time based
assumption later.

---

## Seeding the knowledge model

**Decided: seed manually from the start.** Type in topics with rough levels on day one, the
system corrects from there. Faster to value than starting empty.

The seed splits across two stores, and that split is what makes it safe:

- **Which topics matter** is profile. Adam decides, evidence does not get a vote.
- **How well he knows them** is knowledge model, written as semantic memory carrying a high
  initial confidence. His seed is a starting prior and evidence is allowed to move it.

Justification: self assessed skill level is one of the few places where people are reliably
miscalibrated, so a stale self assessment gets corrected rather than locked in.

This is not an exception to how authority works, it is a consequence of the two halves living
in different stores. See `trust-model.md`.

---

## Manual and conversational memory entry

Wanted, and closer than it looks. Adam wants to be able to say something in plain language
("I did this course three years ago, it covered X and Y") and have it become memory.

That is a text box, a parse into proposed items, and the review queue confirming them.
Since the review queue is already being built for consolidation proposals, this path is
mostly free once it exists. It is not a large later project, and routing it through review
means it reuses the existing write path rather than adding a new one.

---

## Concrete claims, first pass

Rough, from what is currently known. Format is claim, then the output that changes.

**Stable identity, manual**

- Optimizes for depth of understanding over shipping speed → how much explanation
  accompanies any answer, session design
- Primary direction backend and system architecture; secondary embedded and low level, and
  AI systems → news ranking and topic suggestion, with weights
- Values long term quality over short term speed, clarity over cleverness → which
  implementation approaches get proposed at all
- Wants to be known for well structured thoughtful systems → which projects and directions
  get suggested

**Knowledge model, seeded then inferred**

- Competent: C#, .NET, Unity → skip fundamentals, rank beginner content down
- Actively learning: hexagonal architecture, Python with strict typing → surface
  intermediate rather than beginner, go deeper than default
- Touched shallowly: Temporal, LangGraph → sessions can assume vocabulary but not depth
- Not pursuing right now: frontend design → rank down without concluding dislike

**How to work with Adam, manual then refined**

- Mental model first, then structure, then code
- Pair every recommendation with the question, never open ended
- Push back rather than agree
- No speculative abstraction without current pressure
- Concise prose, no hyphens or en dashes or underscores

**Decision records, promoted from features**

- The brain is its own service, because splitting memory across two languages failed in v1
- The brain does not read feature databases, because it couples to internal schemas

Note: the picture of Adam's actual competencies is thin, built only from Notion. The manual
entry path above is how that gets filled in over time.

---

## The practical observation

Most of the highest value entries are **manual**. Stable identity and working preferences
are typed into a profile page. No inference, no consolidation, no model call.

So the first useful version of the brain is a profile page plus a retrieval call, and it
already improves every feature that talks to it. Inference is what makes it compound, but
it is not where the initial value comes from.

That fits the first milestone almost exactly, and it means there is something worth using
well before the hard parts work.

---

## How level is represented

Level is the knowledge model's core value: solid, actively learning, touched shallowly,
deliberately not pursuing. How it is stored is chosen when the schema is written, and that is
the next thing this document needs, because the first milestone seeds the knowledge model by
hand and cannot do that without a shape to seed into.

The candidates are discrete bands, a number, or something else. Whatever is chosen has to work
with seeding above, where the owner's self assessment is a prior that evidence may move, so it
needs to support a value that shifts in small steps. It also has to avoid claiming a precision
nobody can justify, which is the argument that already killed the weighted relevance formula and
the decay curves in `trust-model.md`.
