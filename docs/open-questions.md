# Open questions

Living list. Things we know we need to answer, why they matter, and what has to happen
before we can. Resolved items move to the bottom with the answer, so we can see what was
decided and when.

Decisions already made live in `architecture-decisions.md`. This file is only what is still
open.

Last updated: 2026 08 04

---

## Brain and memory

These are the ones that make or break the platform over years. Treat as the priority track.

### Q1. What goes in a memory event

**Status:** largely answered. Both halves written up, see `brain-knowledge-model.md` and
`brain-event-model.md`. Three smaller threads remain, broken out as Q15 and Q17 below, and
Q16 which is resolved.

Everything downstream depends on it. Consolidation can only find patterns in what was
recorded. Too coarse and there is nothing to learn from, too fine and it is noise that
buries the signal.

**Approach:** work backwards. Establish what the brain should know, then what evidence
would justify it, then what event carries that evidence. Taxonomy becomes an output rather
than a guess, and the same list drafts the first evaluation cases.

**Done (2026 08 01):** the six categories of thing the brain should know, their weighting,
the filter test for what gets in at all, the promotion rule for features, the claim versus
record distinction, the two timestamp requirement, and the decision to seed the knowledge
model manually. Written up in `brain-knowledge-model.md`.

**Correction to an earlier assumption:** this was listed as blocked on the feature list. It
is not. There is a core set of things the brain should know that is feature independent,
plus a lot Adam fills in by hand. Each feature decides what it contributes when it is built,
which means the event schema needs to be extensible by construction rather than complete up
front.

**Done (2026 08 01), second half:** the six event kinds, the event shape, the decision to
use one table for all events, verbs and their declared evidence strength, granularity as a
per feature decision, the topic vocabulary model, relation kinds, and consolidation rules.
Written up in `brain-event-model.md`.

**Remaining threads**, all smaller and broken out separately: Q15 (immediacy scope) and
Q17 (how level is represented). Q16 was resolved the same day, see Resolved.

### Q2. The trust model

**Status:** open, and not blocked on anything. Good candidate for next.

Sub questions:

- How does confidence move as evidence accumulates, and what makes it fall
- What decays, how fast, and does decay differ by memory type
- What happens when explicit profile truth and observed behaviour disagree
- What applies silently versus what waits for approval
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
which go to the review queue.

### Q3. The retrieval interface

**Status:** open, better answered after Q1.

What does a caller actually ask for, and what comes back. v1's single magic call taking an
arbitrary task string was rejected, so the replacement needs designing. Probably several
narrow calls rather than one broad one, but the shape depends on what callers turn out to
need.

### Q4. Review queue lifecycle

**Status:** open, low risk.

Proposal arrives, Adam accepts, rejects, or edits. What happens to each. Rejections are
kept as evaluation data, so the lifecycle has to preserve them rather than delete.

### Q5. Does document memory survive

**Status:** open, small.

It was dropped from the memory layers on the grounds that the wiki covers long form
artifacts. That may not hold for documents that are not about a code project. Worth a
second look before the schema is written.

### Q15. Scope and confirmation on immediate writes

**Status:** open. Deliberately parked, likely answered per feature.

Assertions write immediately rather than waiting for consolidation, which is settled. What
is not settled is scope and confirmation.

**Scope.** If Adam says "be more concise" during a learning session, does that apply to
learning or to everything. Getting this wrong silently is how a system develops habits
nobody asked for. Current lean: default style preferences to global, let Adam narrow, and
have the system state which scope it applied so it can be corrected.

**Confirmation.** Adam wants a prompt when the system learns something, so he can decide.
Some answered inline as a popup, some landing in the review queue, depending on what it is.
Which kind goes where is probably a per feature decision, made when designing what that
feature can learn from an interaction.

**Note, and this is settled:** writing directly does not mean skipping consolidation.
Anything written immediately is still picked up on the next consolidation pass to be linked,
merged, or checked for contradiction. Immediate effect and later integration are separate
concerns and both happen. More checks will be added here as the system grows.

### Q17. How the knowledge model represents level

**Status:** open.

Discrete bands (competent, learning, touched, not pursuing), a number, or something else.
Interacts with the seeding exception in `brain-knowledge-model.md`, where Adam's self
assessed level is a prior that evidence is allowed to move.

### Q6. What consolidation actually does in v1 of the brain

**Status:** open, comes after Q1 and Q2.

The deterministic rules first, LLM reasoning second question. How much can be done without
a model at all.

---

## Architecture and infrastructure

Smaller, and none of them block the brain work.

### Q7. Core in Go or C#

They push the design in different directions. C# invites a richer domain model and heavier
features. Go invites small boring services and pushes complexity into the brain instead.
Neither is wrong but they compound differently over years.

### Q8. What the feature manifest contains

How a feature registers itself: routes, read API version, entity types it exposes, scopes
it requests. Needed before the second feature exists, not before the first.

### Q9. Routing implementation

Path based routing to different processes may be a Caddy or Traefik config file rather than
code, which would leave only a registry and token verification as an actual service.

### Q10. Does projects live in the core codebase

Both are C# so it is tempting. Argument against: the core does auth and routing and must
never be down, and a bug in a scrum board should not take out routing for everything.
Low stakes either way, extraction later would be mechanical.

### Q11. Temporal

Deferred, not rejected. Might earn its place for the consolidation schedule and human in
the loop pauses. Some features might run their own jobs instead. Re decide when there is
something real to orchestrate. Nothing being built now assumes its absence, since
consolidation is idempotent by decision.

---

## Product and scope

### Q12. Shell UX ambition

Unified product, launcher plus dashboard, or thin index. Deliberately left open, since it
constrains nothing else. Worth deciding before any frontend work starts.

### Q13. The feature list

Known so far: wiki, projects, learning sessions, news, reminders and notifications as a
delivery capability. Almost certainly incomplete, and Q1 would benefit from more of it
being on the table.

### Q14. Mobile

Deferred entirely. Swift versus Expo, decided much later, and partly determined by whether
the web frontends end up sharing enough to be worth reusing.

---

## Resolved

Moved here with the answer and the date, so the reasoning is not lost.

### Q16. Do decisions get a first class side table

**Resolved 2026 08 01: no.**

The case for one looked real, since decisions have a lifecycle (active, then superseded)
and an append only event log cannot carry changing status. But the framing was wrong. A
decision is a thing that happened plus a thing with current status, which is exactly the
shape of a semantic memory, and memory items are already separate from memory events.

So the event records that a decision was made, and a memory item holds its current state.
Decisions were never structurally special. Detail in `brain-event-model.md`.

This also corrected an error: decisions were described as pure records that never decay.
A decision produces a permanent record *and* a standing claim that can be superseded.

---

*Decisions made before this file existed are in `architecture-decisions.md`.*
