# Open questions, brain

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

Questions answered inside the brain. Treat this as the priority track: these are the ones that
make or break the platform over years.

Questions whose answer needs the core as well live in `High-level/docs/open-questions.md`.

Resolved items move to the bottom with the answer and the date.

---

## Open

### Q1. What goes in a memory event


**Status:** largely answered. Both halves written up, see `knowledge-model.md` and
`event-model.md`. One smaller thread remains, Q17 below. Q15 and Q16 are resolved.

Everything downstream depends on it. Consolidation can only find patterns in what was
recorded. Too coarse and there is nothing to learn from, too fine and it is noise that
buries the signal.

**Approach:** work backwards. Establish what the brain should know, then what evidence
would justify it, then what event carries that evidence. Taxonomy becomes an output rather
than a guess, and the same list drafts the first evaluation cases.

**Done (2026 08 01):** the six categories of thing the brain should know, their weighting,
the filter test for what gets in at all, the promotion rule for features, the claim versus
record distinction, the two timestamp requirement, and the decision to seed the knowledge
model manually. Written up in `knowledge-model.md`.

**Correction to an earlier assumption:** this was listed as blocked on the feature list. It
is not. There is a core set of things the brain should know that is feature independent,
plus a lot Adam fills in by hand. Each feature decides what it contributes when it is built,
which means the event schema needs to be extensible by construction rather than complete up
front.

**Done (2026 08 01), second half:** the six event kinds, the event shape, the decision to
use one table for all events, verbs and their declared evidence strength, granularity as a
per feature decision, the topic vocabulary model, relation kinds, and consolidation rules.
Written up in `event-model.md`.

**Remaining thread**, broken out separately: Q17 (how level is represented). Q16 was resolved
the same day and Q15 on 2026 08 04, both in Resolved.

### Q2. The trust model


**Status:** in progress. Four of six sub questions settled 2026 08 04, see the settled notes
below.

Sub questions:

- ~~How does confidence move as evidence accumulates, and what makes it fall~~ settled, D26
- ~~What decays, how fast, and does decay differ by memory type~~ settled, D26
- ~~What happens when explicit profile truth and observed behaviour disagree~~ settled, D23
- What applies silently versus what waits for approval — settled for procedural rules
  (D25), still open for knowledge model corrections
- What can never be inferred at all
- How duplicates get detected and merged

Why it matters: a brain that quietly drifts into believing wrong things about you is worse
than no brain. This is the difference between a system you trust in year three and one you
stop opening.

Known starting point: v1's authority ordering (explicit user input outranks confirmed
suggestion outranks repeated evidence outranks inference) is sound. The fixed decimal
weights attached to it were false precision and are dropped.

**Settled 2026 08 04, while working this question:** authority follows the store rather than
the source, see D23. That resolves the third sub question for the profile case: profile and
observed behaviour cannot disagree in a way that needs resolving, because evidence never
moves profile and can at most raise the question with Adam. The knowledge model is where
disagreement is real, and it is resolved by evidence. This also corrected D11, which had
listed skill levels as profile.

Still open within this sub question: which knowledge model corrections apply silently and
which go to the review queue. Lean is more through the queue at first, relaxed over time.

**Also settled 2026 08 04:** scope, which the trust model needs for conflict resolution. Two
dimensions, topic and entity, never feature. Global is the empty set. Scopes form a partial
order, so conflict has three outcomes rather than two and the incomparable case must be
surfaced rather than ranked. See D24. Q15 closed as a consequence, see Resolved.

**Also settled 2026 08 04:** the first two sub questions, confidence movement and decay.
Confidence, currency and status are three separate things that the docs were all calling
decay. Confidence and currency are computed at read time from the evidence links and neither
is stored, so nothing drifts from the evidence behind it and no background job rewrites
memory. They combine per query rather than pre merged, because a question about now and a
question about history want opposite weightings. Currency is undefined for records rather
than slow. Decay curves are deliberately not chosen yet, on the same false precision grounds
that killed the weighted relevance formula. See D26.

Nothing is ever archived for being old or uncertain. Consolidation never retires anything;
only Adam does, and retirement means inactive rather than destroyed. Editing a derived claim
transfers provenance to Adam, and every edit or retirement records a judgment event so
consolidation does not re propose what he just rejected. See D22 and D27.

**Still open in Q2:** how duplicates get detected and merged, what can never be inferred at
all, and which knowledge model corrections apply silently rather than through the queue.

Two of those three are consolidation side, so they belong with Q6 rather than being answered
before consolidation exists.

### Q4. Review queue lifecycle


**Status:** open, low risk.

Proposal arrives, Adam accepts, rejects, or edits. What happens to each. Rejections are
kept as evaluation data, so the lifecycle has to preserve them rather than delete.

### Q17. How the knowledge model represents level


**Status:** open.

Discrete bands (competent, learning, touched, not pursuing), a number, or something else.
Interacts with seeding, where Adam's self assessed level is a prior evidence may move.

**Now blocking:** milestone one seeds the knowledge model by hand, so this has to be answered
before that schema is written. It moved from a loose end to the next thing needed.

### Q6. What consolidation actually does in v1 of the brain


**Status:** open, comes after Q1 and Q2.

The deterministic rules first, LLM reasoning second question. How much can be done without
a model at all.

**Inherited from Q3 on 2026 08 04:** whether "evidence exists but no claim has formed yet" is
a state retrieval can express, or whether silence is fine. It looked like a retrieval question
and is not. It is a question about what consolidation does with weak evidence, and it does not
arise until consolidation exists.

---

## Architecture and infrastructure

Smaller, and none of them block the brain work. One of them now sits on the core track instead,
which D18 puts first: Q9 shapes how much of the core is actually code.

---

## Also constrained by

Open elsewhere, but blocking or shaping brain work.

- **Q11**, Temporal, in `High-level/docs/open-questions.md`. Would orchestrate the
  consolidation schedule if it earns its place.

---

## Resolved

### Q5. Does document memory survive


**Closed 2026 09 03: it does not, and the question went with it.** It only existed because
document memory had been dropped on the grounds that the wiki covers long form artifacts, and
the wiki may never be built, so the justification was the thing that failed rather than the
conclusion.

Document memory stays out of the brain. If a feature ever needs long form artifacts remembered,
that is raised then, with that feature's actual needs rather than a guess about the wiki's.

### Q3. The retrieval interface


**Resolved 2026 08 04**, to the depth needed to build. The remaining detail is deliberately
unwritten until there is a feature to shape it.

**What v1 got wrong was genericity, not specificity.** One call served every caller, which is
why it needed a provider interface plus a switch on workflow type. Named per feature calls
delete the switch by construction. The response being nine kinds of thing was not the problem
either; every caller getting the same nine was.

**The answer:**

- Named per feature calls facing out, in that feature's thin module in the brain, composed from
  a small set of shared primitives facing in. Features never see the primitives and never merge
  anything themselves.
- The brain returns what it knows about Adam bearing on the task. The feature decides what to
  do about it. The brain never returns "teach this next", because that is the feature's domain.
- Calls are bespoke, items are not. Consistent item shape, with evidence references included by
  default so "why do you believe this" is answerable at the point of use.
- A call declares now versus history, since that changes how currency is weighted.
- Conflicts are returned as conflicts, never silently resolved.

**Milestone consequence:** D18 has no features, so it builds the primitives only and the
evaluation harness is the first caller. Everything retrievable at that point is manually
entered, since consolidation does not exist yet. Per feature calls arrive with the first
feature.

Detail in D28.

### Q15. Scope and confirmation on immediate writes


**Resolved 2026 08 04: the question dissolved rather than being answered.**

It asked two things. Whether "be more concise" said during a learning session applies to
learning or everything, and whether the system should prompt when it learns something.

Both existed only because D21 let a single in conversation utterance create a standing global
rule immediately. That was generalising from one data point, which the rest of the design
refuses to do. Removing it removes the question:

- An instruction mid conversation is temporary, applies to that conversation only, and is
  conversation state rather than memory. It is recorded as an event so consolidation can use
  it later. See D25.
- A standing rule exists only because Adam typed it into the rules list. It applies
  immediately because he put it there, and there is nothing to confirm.
- Scope was only hard for rules born from conversation. Those no longer exist. Procedural
  memory is global by construction. See D24.

The channel resolves what no amount of structure could: said in conversation means temporary,
typed into the rules list means standing.

Accepted cost: the brain will not learn working style passively. Consolidation proposing rules
from repeated occurrences is deferred, and is a stronger basis than one sentence anyway.

### Q16. Do decisions get a first class side table


**Resolved 2026 08 01: no.**

The case for one looked real, since decisions have a lifecycle (active, then superseded)
and an append only event log cannot carry changing status. But the framing was wrong. A
decision is a thing that happened plus a thing with current status, which is exactly the
shape of a semantic memory, and memory items are already separate from memory events.

So the event records that a decision was made, and a memory item holds its current state.
Decisions were never structurally special. Detail in `event-model.md`.

This also corrected an error: decisions were described as pure records that never decay.
A decision produces a permanent record *and* a standing claim that can be superseded.

---

*Decisions made before this file existed are in `High-level/docs/platform-architecture.md`.*
