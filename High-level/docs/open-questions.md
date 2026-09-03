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
