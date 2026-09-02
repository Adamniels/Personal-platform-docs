# Open questions, platform wide

Status: exploratory. Nothing is built.

Last updated: 2026 09 02

Living list. Questions live here when **answering them requires knowing about more than one
area**, or when they are about the platform rather than any one service. A question that merely
affects several areas but is answered inside one of them lives with that one.

Resolved items move to the bottom with the answer and the date.

- `Core/docs/open-questions.md` — questions answered inside the core
- `Brain/docs/open-questions.md` — questions answered inside the brain

---

## Cross cutting

### Q18. How user isolation is enforced


**Opened 2026 09 02, from D29.** Every table in the brain carries a user id, so one forgotten
`WHERE` clause is a cross user data leak rather than a wrong answer.

The two options are query layer discipline, where correctness rests on review, and Postgres row
level security, where the database refuses to return another user's rows regardless of what the
query says. Leaning strongly toward row level security, since it converts a class of bug into
something structurally impossible, and since the security work is a goal of this project rather
than an overhead. Settle it when the core and the brain schema are planned in depth, not before.

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
