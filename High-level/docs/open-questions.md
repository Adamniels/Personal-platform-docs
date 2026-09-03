# Open questions, platform wide

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

Living list. Questions live here when **answering them requires knowing about more than one
area**, or when they are about the platform rather than any one service. A question that merely
affects several areas but is answered inside one of them lives with that one.

Resolved items move to the bottom with the answer and the date.

- `Core/docs/open-questions.md` — questions answered inside the core
- `Brain/docs/open-questions.md` — questions answered inside the brain

---

## Infrastructure

### Q11. Temporal


Deferred, not rejected. Might earn its place for the consolidation schedule and human in
the loop pauses. Some features might run their own jobs instead. Re decide when there is
something real to orchestrate. Nothing being built now assumes its absence, since
consolidation is idempotent by decision.

### Q19. What language the notification service is written in


**Opened 2026 09 02, from D30.** It is scheduling and delivery, nothing AI shaped, and it holds
no memory. Rust would fit and would extend the practice motive behind D14. Deliberately not
decided now, since nothing depends on it and D14 no longer pre assigns languages to services
that have not been designed.

---

## Product and scope

### Q12. Shell UX ambition


**Narrowed 2026 09 03.** The architecture half is decided in D34, which was the half that
constrained anything. This question previously claimed it constrained nothing else, which was
false: it decided whether the core grows, whether the brain becomes a dashboard proxy, and
whether features have to expose UI pieces.

What is left is how rich the shell is. A list of links, a front page with real content on it, or
one unified product where every feature shares a design system and navigation feels seamless.

Deliberately still open, and this half genuinely cannot be reasoned out. It is answered by using
the thing. Two things are worth carrying into that: the stated ambition of a main daily tool
makes the list of links unlikely, and the unified product is the largest permanent cost in the
whole plan, larger than D14's polyglot cost, because every feature ever built has to be dragged
into the same design system forever.

### Q14. Mobile


Deferred entirely. Swift versus Expo, decided much later, and partly determined by whether
the web frontends end up sharing enough to be worth reusing.

---

## Resolved

Moved here with the answer and the date, so the reasoning is not lost.

### Q13. The feature list


**Closed 2026 09 03. It was not a question.** It asked for the complete list of features while
admitting in the same sentence that any answer would still be incomplete, which is a note
wearing the shape of a question. A platform built over years has no complete list, so nothing
could ever have resolved it.

What it was carrying is a set of candidates worth designing with in mind, none of them
committed. Those are in `High-level/docs/platform-architecture.md` under "Candidate features".

Its stated justification was stale as well. It said Q1 would benefit from more of the list on
the table, but Q1 had already overturned that: the event schema was made extensible by
construction precisely so the list would stop mattering.

One thing did get settled while closing it, and Q12 needs it: the platform's intended size. See
"What this is" in `platform-architecture.md`.

### Q18. How user isolation is enforced


**Resolved 2026 09 03: enforced by the datastore, platform wide. See D33.**

It was opened leaning this way already, so the value was in what changed while settling it
rather than in the choice.

Its framing was too narrow. It said "every table in the brain", but features hold per user data
too, and a leak in the wiki is exactly as bad as one in the brain. The answer had to reach
further than the question did.

And it is stated as a property rather than a mechanism, which is what lets it bind a feature on
a datastore nobody has picked yet without dictating what that datastore is.

The reason to settle it now rather than at schema time, as this question originally said, is
that D18 started the core track, so schema time arrived.
