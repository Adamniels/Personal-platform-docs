# The trust model

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

Who is allowed to change what, when a written thing takes effect, and how belief is scored and
corrected. This is the densest cluster in the design and the six decisions below only make
sense together.

Two of its rules live upstream because they constrain features rather than the brain: D20
observation granularity, and D21 records apply immediately while claims wait. Both are in
`High-level/docs/feature-contract.md`.

---

### D22. Nothing is destroyed, and only the owner retires


Nothing is ever deleted. An item can become inactive, meaning it stops applying but stays
visible in history and can be brought back.

Two causes, and no third:

- something superseded it, expressed as a relation
- the owner retired it by hand

Never age, never a low score, never a consolidation decision. **Consolidation never retires
anything.** It proposes, it creates supersedes relations, and it lets scores fall out of the
evidence. Putting something away is the owner's action.

The owner can retire or edit anything in their own brain, not only rules. That manual override is what
makes the rest of the trust model safe: whatever the system concludes, there is a place to go
and correct it.

Related rule: consolidation proposes, it never silently rewrites.

**Amended 2026 08 04.** This previously read "consolidation prunes by archiving". The
archiving was never the problem, the actor was. Pruning implies putting things away for being
old or low value, which is the path by which a permanent record could quietly go invisible.
Age and confidence are ranking inputs only. See D26.

comment to remove after we gone through this: really have to think what a session means and how we keep track of that, it might be up to each
feature to keep track of it, like learning sessions, it should hold every time I work with that
session even if I opened a new chat in it.

### D23. Authority follows the store, not the source


"Explicit user input outranks inference" was doing too much work. It holds for preferences
and fails for self assessment, and reading it as one rule is what put skill levels inside
profile, where they could never be corrected.

Authority is a property of what a claim is about, and that is already expressed by which
store the claim lives in:

| Store            | What it holds                                                  | Who can change it                                                      |
| ---------------- | -------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Profile          | identity, goals, values, interests, preferences, working style | The owner only. Evidence never moves it, it can at most raise the question |
| Knowledge model  | what the owner knows and how well                              | The owner seeds it, evidence moves it, silently or through the review queue |
| Reported history | what the owner says happened                                   | The owner only, since nothing else can confirm or contradict it        |

So there is no authority ordering to apply at read time and no exception to carve out. Where
a claim lives tells you who wins. The v1 ordering survives only inside a single store, for
ranking evidence of the same kind against itself.

Consequence: seeding the knowledge model writes semantic memory carrying a high initial
confidence, not profile. That is why the seed is movable, and it stops being a special case.

Which knowledge model corrections happen silently and which go to the review queue is settled
when consolidation is built, since consolidation is what produces them. The lean is that more
goes through the queue at first, relaxed over time as it becomes clear what is safe to apply
silently.

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

A global procedural rule exists only because the owner added it to the rules list on purpose.
Because he typed it there, it applies immediately and there is nothing to confirm. Global means
global to that user, see D29.

Why this rather than inferring the rule from the utterance: the channel resolves an ambiguity
that no amount of structure could. Said in conversation means temporary, typed into the rules
list means standing. No default to choose, no scope to announce, no confirmation step. This is
what dissolved a cluster of scope and confirmation problems that existed entirely to manage the
consequences of the opposite choice.

**The temporary instruction is still recorded as an event.** It changes nothing permanent, but
it is the evidence consolidation needs later to propose the standing rule, and it cannot be
backfilled. Same argument that already justifies recording activity uninterpreted.

Accepted cost, stated plainly: the brain does not pick up working style passively. The owner
has to notice he is repeating himself and add the rule. The eventual fix is consolidation proposing a
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
- **Status.** Active or inactive. Discrete, and set only by supersession or by the owner, see D22.

**Confidence and currency are both computed at read time from the evidence links. Neither is
stored.** No background job walks the table lowering numbers, since that is consolidation
quietly rewriting memory, which is already forbidden. Nothing can drift from the evidence
behind it, and "why do you believe this" is answered by the same structure that produced the
score. Verbs already declare strength and polarity, so the inputs exist. Every read is scoped
to one user, so the read cost is nothing regardless of how many accounts exist, see D29.

**They combine into one score, but the weighting depends on the question.** "What is Adam into
now" wants currency to dominate. "What has Adam worked on over the years" wants it ignored
entirely, because applying it there would hide the answer. That is why they cannot be stored
pre merged, and it means a retrieval call has to say whether it is asking about now or about
history. Constraint on the retrieval interface, see D28.

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


**Editing a derived claim makes it the owner's.** Provenance changes from derived to asserted,
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
because the owner is definitionally correct there. Rejecting a knowledge model claim is strong
negative evidence that later evidence may still move, because that is the store where he is a
fallible witness. No new rule needed.

Product consequence: being able to edit anything means the memory centre is a general item
browser rather than a profile page plus a rules list. Architecturally free, but a meaningful
share of the real build work sits there, and it is larger than what D18 scoped for the first
milestone.

---

## Duplicates

How duplicates are detected and merged is consolidation's work and is written when consolidation
is. It belongs to the trust model because merging is the one operation that can lose an evidence
link, and D22 forbids destroying anything. So a merge is a supersedes relation with both sides
still readable, not a rewrite that leaves one claim standing.

## What is never inferred

Some things should not be concluded about a person even where the evidence supports them.
Profile is already protected, since under D23 evidence cannot move it at all. What is left is the
knowledge model and behavioural patterns, and the live case is the creepiness problem flagged
under Behavioural patterns in `knowledge-model.md`. The line is drawn when consolidation exists
and there is something concrete to refuse, rather than as a list written in advance.

## The review queue

A proposal arrives carrying its evidence and its confidence, and the owner accepts, rejects, or
edits it. The three do different things, and D27 already fixes two: an edit transfers provenance
to the owner, and both an edit and a rejection record a judgment event so the next pass does not
re propose what was just refused.

**Rejections are kept, never deleted.** Every accept and every reject is a labelled evaluation
example, which D16 wants and which cannot be reconstructed later. So the lifecycle has to
preserve them.

The rest, what a proposal looks like while it waits and what happens to one nobody ever answers,
is written when the queue is built.
