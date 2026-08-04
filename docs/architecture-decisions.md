# Personal platform v2, architecture decisions

Status: exploratory. Nothing is built. This records what has been decided in discussion,
why, and what is still open, so we stop relitigating settled ground.

Last updated: 2026 08 04

---

## What this is

The second version of a personal platform. A stable core plus features that are built one
at a time, independently. A shared brain (memory) is the thing that makes it a platform
rather than a collection of apps, and it is the part most worth getting right.

Related Notion pages: "Northstar OS" (project entry), "Northstar OS, my personal platform",
"Wiki project". v1 material lives under "My Platform (1)" and "Implementing memory" and is
reference only, see "Carried from v1" below.

**Other documents in this folder**

- `open-questions.md` — living list of what is still undecided, and what blocks each
- `brain-knowledge-model.md` — what the brain is trying to know about Adam
- `brain-event-model.md` — how evidence gets recorded, and the event shape
- `overview.html` — synthesis of the whole design state in one page, with a diagram. Derived
  from the files above rather than authoritative, so regenerate it when decisions change

---

## The framing

Two questions that kept getting bundled together, and must stay separate:

1. **Where does a feature's UI and data live?** Answer: isolated. Own frontend, own
   backend, own database.
2. **What is a feature allowed to depend on?** Answer: a small fixed set of platform
   contracts, and nothing else.

Isolation of implementation, coupling through contracts. Almost every decision below
follows from holding those two apart.

---

## Decisions

### D1. The brain is its own service

It owns three things: memory, cross feature brokering, and cross feature entity links.
It has its own database.

Rationale: if memory lives inside a general backend, that backend slowly absorbs every
feature's data because it is always the path of least resistance. Making the core a client
of the brain rather than its owner keeps that honest.

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

### D7. The brain never makes a decision that belongs to a feature's domain

It routes, merges, ranks, and remembers. It does not decide what makes a good wiki node or
when a project task is blocked. Ranking and merging are its own domain, so those are not
violations.

### D8. Each feature wraps brain access behind one internal port

Not scattered HTTP calls through the codebase. One adapter.

Note the justification changed during discussion. This is **not** for portability or future
extraction, see D9. It is for testability, so a feature can run against a fake brain in its
own test suite, and so the implementation can be swapped without touching callers.

### D9. We are not designing for standalone extraction

The brain is a hard dependency. A feature may run without it but will not be useful without
it. If a feature ever needs to become a standalone product, a brain equivalent gets written
at that point.

Consequence, and it simplifies a lot: no graceful degradation paths, no fallback adapters,
no speculative portability seams. Features assume the brain is there.

### D10. Feature tiers

| Tier | Owns a database | Owns a frontend | Talks to the brain |
|---|---|---|---|
| Federated | yes | yes | yes, both directions |
| Embedded | no, stores in the brain | rendered by the core shell | yes, it is inside it |
| Linked | irrelevant | its own | no |

The tier mostly decides itself: **a feature written in a language other than the brain's is
federated by construction**, because embedded means living inside the brain's process and
database. Anything owning real domain data is federated too.

Current placement: wiki federated, projects federated, learning federated,
reminders is not a feature at all, see D13.

### D11. Profile lives in the brain

Identity, goals, values, interests, preferences, and how Adam wants to be worked with.
Explicit user entered facts, and any context read has to merge them with inferred memory.
Splitting them across services means a network join on every read.

Line to hold: the core owns identity, who you are as an account. The brain owns who you
are as a person.

**Corrected 2026 08 04.** An earlier version listed skill levels as profile. They are not.
Profile holds the things Adam is definitionally correct about, so nothing in it moves except
by his own hand. Skill level is a self assessment, which he can be miscalibrated about, so it
lives in the knowledge model and evidence is allowed to move it. See D23.

### D12. Cross feature entity links live in the brain, generically

Stored as: this entity in feature A relates to that entity in feature B. Created by hand,
by Adam, through a feature's UI which asks the brain what could be linked rather than
asking the other feature.

The only concrete case today is a platform project that also exists in the wiki. The
generic form is chosen not because reuse is expected but because the alternative is a
wiki id column on the project table, which breaks D3. Consistency, not anticipated reuse.

### D13. Reminders and notifications are a delivery capability, not a feature

A feature sends a request: tell Adam this, at this time or under this condition. Nothing
gets linked. Standing self authored reminders ("use TDD", "plan before prompting") go
through the same service with Adam as the source instead of a feature.

This corrected an earlier misclassification of reminders as an embedded feature.

### D14. The stack is three languages

- **Python** for anything AI shaped: the brain, wiki, learning, news.
- **C#** for CRUD heavy and infrastructure pieces: projects, possibly the core.
- **TypeScript** for frontends, likely React.

Mobile is deferred entirely, Swift or Expo, decided much later.

The specific argument for the brain being Python: its hardest work is consolidation,
embeddings, clustering, and LLM orchestration, and v1's real failure was splitting memory
across two languages. One service in the language where its hardest work lives removes that
split by construction.

Cost of going polyglot, accepted knowingly: separate builds, separate tooling, no shared
types, and contract drift caught at runtime. Mitigation is schema first contracts
(OpenAPI or protobuf) with generated clients on both sides, committed to from day one.

### D15. No LLM calls on the read path

Reads are SQL plus vector, deterministic, tens of milliseconds. All expensive reasoning
happens in consolidation, offline. This is what makes coupling every feature to one shared
brain safe on latency grounds.

Related: a feature fetches a context packet once per run, not per page render. Writes are
events, fire and forget, never blocking a user action.

**Amended by D21.** An earlier version of this said to cache the profile snapshot for
latency. That breaks immediacy, since a directly written assertion has to affect the very
next request. Caching is fine but must invalidate on write, which is trivial for one user.

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

### D18. First milestone

Profile, plus events, plus deliberately dumb retrieval, plus roughly ten evaluation cases.
Nothing else. Small, but it makes everything after it measurable.

Note added 2026 08 04: D27 lets Adam edit and retire anything in the brain, which makes the
memory centre a general item browser rather than a profile page. That is not first milestone
work, but it is where a meaningful share of the real build sits, so the milestone stays honest
only if the browser is understood as coming later rather than being implied by "profile".

### D19. One table for all events, with an extensible schema

Every event, from every feature, in one table. Structure is a verb, a subject, a context,
two timestamps (when it happened and when it was recorded), and a feature specific payload.

Rationale: consolidation can then reason over events from features that did not exist when
it was written. Per feature tables would mean teaching the consolidator about every new
feature. Scale objections are irrelevant at single user volume.

Feature facing consequences:

- **Features may add verbs, but a new verb must declare its evidence strength and
  polarity.** A verb the brain cannot weigh is unusable.
- Every event carries a payload version and an idempotency key.
- Relation types are likewise extensible. A fixed enum will be wrong within a year.

Detail in `brain-event-model.md`.

### D20. Observation granularity is a per feature decision

Not a global rule. How finely a feature observes is decided when that feature is designed,
since it depends on what the feature does and what the brain should learn from it. The
brain owns the verb vocabulary so anything emitted can be weighed; the feature owns how
finely it looks.

Ruled out globally: interaction level tracking (scroll position, time on screen). Needs
instrumentation everywhere and time on screen is a bad proxy.

### D21. Records apply immediately. Claims wait for consolidation

Consolidation exists to infer claims from behaviour. When Adam records a fact there is
nothing to infer, so it does not wait.

- **Records write immediately.** A profile edit, "I took that course in 2023", "this session
  was too basic". Each states a fact, so there is nothing to get wrong.
- **Exposure, engagement and activity** are evidence. Generalizations from them wait.
- **Features write records directly. Features never write claims directly.** A claim about
  Adam goes through evidence and consolidation.

**Writing immediately does not mean skipping consolidation.** Anything written directly is
still picked up on the next pass to be linked, merged, or checked for contradiction.
Immediate effect and later integration are separate concerns and both happen.

**Amended 2026 08 04.** This was titled "assertions apply immediately" and used "talk to me
more concisely affects the next message" as its example. The example was wrong, and it was
the source of a great deal of accidental complexity. "Adam wants concise prose" is a claim
about Adam, not a record, and letting one sentence create a standing global rule is
generalising from a single data point, which this document refuses to do everywhere else. An
instruction mid conversation is a claim wearing an imperative. See D25.

### D22. Nothing is destroyed, and only Adam retires

Nothing is ever deleted. An item can become inactive, meaning it stops applying but stays
visible in history and can be brought back.

Two causes, and no third:

- something superseded it, expressed as a relation
- Adam retired it by hand

Never age, never a low score, never a consolidation decision. **Consolidation never retires
anything.** It proposes, it creates supersedes relations, and it lets scores fall out of the
evidence. Putting something away is Adam's action.

Adam can retire or edit anything in the brain, not only rules. That manual override is what
makes the rest of the trust model safe: whatever the system concludes, there is a place to go
and correct it.

Related rule: consolidation proposes, it never silently rewrites.

**Amended 2026 08 04.** This previously read "consolidation prunes by archiving". The
archiving was never the problem, the actor was. Pruning implies putting things away for being
old or low value, which is the path by which a permanent record could quietly go invisible.
Age and confidence are ranking inputs only. See D26.

### D23. Authority follows the store, not the source

"Explicit user input outranks inference" was doing too much work. It holds for preferences
and fails for self assessment, and reading it as one rule is what put skill levels inside
profile, where they could never be corrected.

Authority is a property of what a claim is about, and that is already expressed by which
store the claim lives in:

| Store | What it holds | Who can change it |
|---|---|---|
| Profile | identity, goals, values, interests, preferences, working style | Adam only. Evidence never moves it, it can at most raise the question |
| Knowledge model | what Adam knows and how well | Adam seeds it, evidence moves it, silently or through the review queue |
| Reported history | what Adam says happened | Adam only, since nothing else can confirm or contradict it |

So there is no authority ordering to apply at read time and no exception to carve out. Where
a claim lives tells you who wins. The v1 ordering survives only inside a single store, for
ranking evidence of the same kind against itself.

Consequence: seeding the knowledge model writes semantic memory carrying a high initial
confidence, not profile. That is why the seed is movable, and it stops being a special case.

Left open, and now clearly framed: which knowledge model corrections happen silently and
which go to the review queue. Part of Q2. Current lean is that more goes through the queue at
first, relaxed over time as it becomes clear what is safe to apply silently.

### D24. Scope is topic and entity. Never feature

Scope is not an enum. It is the set of qualifiers on a claim, matched against the situation a
retrieval request already carries. Global is the empty set, which matches everything.

Two dimensions:

- **topic**, from the topic vocabulary
- **entity**, a specific project or artifact

**Feature is deliberately not one.** It describes where a claim fires rather than what it is
about, and it is the dimension that lets contradictory behaviour accumulate quietly: concise
in learning, thorough in wiki, something else in news, and in two years the system behaves
differently in each place for reasons nobody remembers. Topic and entity do not have that
failure mode, because they describe the subject rather than the caller.

Default scope falls out of the claim instead of needing a policy. If a claim names a subject,
that subject is its scope. If it names none, it is global.

Consequences:

- **Procedural memory needs no scope matching at all.** Every rule is global by construction,
  so two rules are always at the same scope, and conflict is supersession plus recency. The
  machinery below only has to exist for claims carrying a subject.
- **Scope comparison is a partial order, not a line.** Three outcomes: one scope is strictly
  narrower and wins regardless of age; scopes are equal and recency wins; scopes are
  incomparable, topic postgres versus entity operation rollout, and neither wins. The third
  must be surfaced as a conflict, never resolved by picking. The obvious implementation
  compares two scopes and returns a winner, and that would be silently wrong.

Known soft spot: nothing structurally prevents a subject being created that is a one to one
proxy for a feature, which quietly recreates what was rejected here. The guard is that it has
to be stated as a subject and shows up in the rules list as one. Discipline, not mechanism.

### D25. Standing rules are created deliberately. Conversation instructions are temporary

"Be more concise", said mid conversation, applies to that conversation or session and nothing
else. It is temporary, it is conversation state rather than memory, and it never becomes a
standing rule on its own.

A global procedural rule exists only because Adam added it to the rules list on purpose.
Because he typed it there, it applies immediately and there is nothing to confirm.

Why this rather than inferring the rule from the utterance: the channel resolves an ambiguity
that no amount of structure could. Said in conversation means temporary, typed into the rules
list means standing. No default to choose, no scope to announce, no confirmation step. This is
what closed Q15, which existed entirely to manage the consequences of the opposite choice.

**The temporary instruction is still recorded as an event.** It changes nothing permanent, but
it is the evidence consolidation needs later to propose the standing rule, and it cannot be
backfilled. Same argument that already justifies recording activity uninterpreted.

Accepted cost, stated plainly: the brain does not pick up working style passively. Adam has to
notice he is repeating himself and add the rule. The eventual fix is consolidation proposing a
rule from repeated occurrences, which is a far stronger basis than one sentence, so the
capability is deferred rather than lost.

Where the temporary preference lives: with whatever runs the conversation. That is working
memory, already dropped from the memory layers as workflow state belonging to the workflow, so
this is that decision holding rather than a new one.

The rules list has to be easy to read, add to, edit and delete by hand. It is the surface that
makes global rules safe.

### D26. Confidence, currency and status are three different things

The docs called all three decay. They are independent and they behave differently.

- **Confidence.** How strongly the evidence supports a claim. Moves when evidence arrives,
  never with time.
- **Currency.** How likely the claim still describes now. A function of time since it was last
  supported.
- **Status.** Active or inactive. Discrete, and set only by supersession or by Adam, see D22.

**Confidence and currency are both computed at read time from the evidence links. Neither is
stored.** No background job walks the table lowering numbers, since that is consolidation
quietly rewriting memory, which is already forbidden. Nothing can drift from the evidence
behind it, and "why do you believe this" is answered by the same structure that produced the
score. Verbs already declare strength and polarity, so the inputs exist. At single user volume
the read cost is nothing.

**They combine into one score, but the weighting depends on the question.** "What is Adam into
now" wants currency to dominate. "What has Adam worked on over the years" wants it ignored
entirely, because applying it there would hide the answer. That is why they cannot be stored
pre merged, and it means a retrieval call has to say whether it is asking about now or about
history. Constraint on Q3.

**Currency is undefined for records, not slow.** A record asserts nothing about now, so the
mechanic does not apply to it. The rule that records must never be downweighted into
invisibility stops being a special case anyone has to enforce and becomes a consequence of
what a record is.

**Currency has a narrower job than it first appears.** "Focused on embedded in 2024" and
"focused on AI from 2025" are sequential rather than conflicting, and are handled by
supersession with the older claim keeping a closed time range. Nothing has to be guessed from
timestamps. Currency is for claims that go quiet with no successor: nobody said Adam stopped,
the evidence simply stopped arriving, and silence is the only signal. Applying currency where
supersession should have been used is the mistake to avoid.

**No decay curves yet.** Half lives per memory type are the same shape as the weighted
relevance formula already rejected for false precision: no ground truth to tune against and no
way to tell whether a change helped. Start by exposing when a claim was last supported, order
by it, flag anything past a crude threshold as worth re checking, and earn the curve through
the evaluation harness.

Accepted costs: there is no cheap answer to "what does the system believe right now" without
computing over the corpus, and materialising that is a cache needing invalidation on write,
the same trap the profile snapshot fell into. And belief becomes a gradient with no natural
cutoff, so rendering a "what I believe about you" list needs a display threshold somewhere.

### D27. Editing changes provenance, and every edit records a judgment

**Editing a derived claim makes it Adam's.** Provenance changes from derived to asserted,
which is what D23 reads to decide who may move it afterwards. Edit a profile item and it is
absolute. Edit a knowledge model claim and it becomes a new prior that evidence may still
move, exactly as seeding does. Without this the claim keeps its old provenance and the system
treats a correction as just another inference it is free to revise.

**An edit or a retirement is recorded as a judgment event against the original.** Otherwise
the supporting evidence is still sitting there, consolidation runs, sees the same pattern and
proposes the same claim again. Retire it, it returns, retire it again. That is the most
irritating failure a memory system can have, and one event prevents it rather than any clever
logic. It also lands the rejection in the evaluation set for free, which D16 already wants.

**Rejection strength follows the store, per D23.** Rejecting a profile claim is absolute,
because Adam is definitionally correct there. Rejecting a knowledge model claim is strong
negative evidence that later evidence may still move, because that is the store where he is a
fallible witness. No new rule needed.

Product consequence: being able to edit anything means the memory centre is a general item
browser rather than a profile page plus a rules list. Architecturally free, but a meaningful
share of the real build work sits there, and it is larger than what D18 scoped for the first
milestone.

---

## The two contracts

Both stay small. If either drifts large, something has gone wrong.

**Downward, what the brain offers a feature**

- get context for a task
- get profile snapshot
- search
- record event
- propose memory candidate
- index document

**Upward, what a feature offers the brain**

- a narrow read API over its own domain, shaped per feature
- the events it emits

Only the brain calls a feature's read API. That is a topology rule rather than a security
one: it is what keeps features from linking to each other directly.

---

## Memory layers

Reduced from v1's seven to four, plus links:

1. **Profile**, identity, goals, values, interests, preferences and working style. Entered by
   Adam, moved only by Adam. See D23 for what is deliberately not in here
2. **Events**, append only record of what happened
3. **Semantic**, claims and records the brain holds about Adam, each linked to whatever it
   rests on. Most are derived from repeated evidence, but not all: the knowledge model's seed
   values, a decision's current status, and reported history are single sourced and live here
   too
4. **Procedural**, versioned rules for how the system should behave
5. **Cross feature links** (not memory, but lives in the brain's database)

Dropped: working memory (it is workflow state and belongs to whatever runs the workflow),
graph memory (never built in v1, and the link table covers the real use case), document
memory (overlaps heavily with what the wiki now does).

Document memory is dropped **provisionally**. The reasoning assumes the wiki covers long
form artifacts, which may not hold for documents that are not about a code project. Tracked
as Q5, and worth a second look before the schema is written.

---

## Considered and dropped

Recorded so we do not re open them without new information.

**The brain connecting directly to feature databases.** Couples the brain to every
feature's internal schema, breaks when a feature refactors, bypasses the feature's own
rules so the brain reimplements them badly, and spreads credentials. Replaced by D4.

**A generic search envelope every feature conforms to.** Designed to avoid touching the
brain when adding a feature. At five features that cost is not real, and flattening every
domain into one shape loses structure that matters. Speculative abstraction, dropped.

**Designing for standalone extraction.** Explicitly not a priority. See D9.

**Splitting memory ownership across two languages.** This was v1's actual failure: the
design said one language owned canonical memory and the other proposed candidates, and the
boundary leaked in practice. Avoided structurally in v2.

**A weighted relevance formula with fixed constants.** v1 had authority times 0.35 plus
similarity times 0.25 and so on. There is no ground truth to tune against and no way to
tell if a change helped. Start with filtering and simple ordering, earn the weights through
evaluation.

**One magic GetMemoryContext call.** v1 had a single method taking an arbitrary task string
and returning nine kinds of thing. It did not fit, and ended up needing a provider interface
plus a switch on workflow type. Let callers ask for what they need.

**The operating system framing.** Northstar OS was a name, not an architecture. Dropped as
a design metaphor.

**A side table for decisions.** They have a lifecycle (active, then superseded) that an
append only event log cannot carry, which looked like it needed special handling. It does
not: a decision is a thing that happened plus a thing with current status, which is exactly
the shape of a semantic memory, and memory items are already separate from memory events.
Resolved as Q16.

**Section level instrumentation as a global requirement.** Considered as the passive signal
that would maintain the knowledge model. Superseded by direct assessment: the quiz Adam
already wanted at the end of learning sessions measures knowledge rather than guessing at
it. Emit section detail where a feature knows its own structure for free; do not build
tracking for it. See D20.

---

## Open questions

Tracked in `open-questions.md`, which is the single list. Keeping a second copy here
guarantees the two drift apart.

---

## Carried from v1

Ideas worth keeping, as ideas, not code. v2 is a fresh build.

- **Events then consolidate.** Features record what happened; a background pass interprets
  it later. Refined in D21: features never write *claims* directly, but records are fine,
  and explicit assertions do not wait for consolidation to take effect.
- **The review queue.** Proposals with evidence and confidence that Adam accepts or rejects.
  This is the trust mechanism and the reason the system stays inspectable.
- **Authority ordering.** Explicit user input outranks confirmed suggestion outranks
  repeated evidence outranks inference. Kept, but demoted by D23: it applies within a store
  rather than across all of memory. The false precision of fixed decimal weights is dropped.
- **The memory centre UI concept.** Profile, learned about me, timeline, rules, review
  queue. A place to see and correct what the system believes.
- **Not markdown as the storage substrate.** Markdown is a fine authoring format for things
  written by hand. It is a bad query substrate. Storage and retrieval is Postgres plus
  pgvector.

One thing v1 wanted and could not do: letting an agent find relevant data itself rather than
being fed it. v2's shape, a brain sitting in front of every feature's read API, is the shape
that makes that possible. Not designed for, but worth noticing.
