# Platform architecture

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

This is the top of the documentation. It holds what you need to understand the system before
building any part of it: what the platform is, what services exist, and the decisions that
constrain more than one of them.

Anything you only need when you sit down to write a particular service lives in that service's
folder instead, `Brain/docs/` or `Core/docs/`. Those documents point back here. This one points
down to them for detail rather than repeating it.

**On numbering.** Decision numbers (D1 to D34) and question numbers (Q1 to Q19) are global
and permanent across every folder. They are never renumbered when a document moves, because
decisions cite each other and several carry amendment history. `docs/README.md` at the repo
root maps every number to the file it lives in.

**Related documents**

- `High-level/docs/feature-contract.md` — what a feature is, and what it may depend on
- `High-level/docs/open-questions.md` — what is still open across the platform
- `Core/docs/` — the core in depth
- `Brain/docs/` — the brain in depth

---

## What this is

The second version of a personal platform. A stable core plus features that are built one
at a time, independently. A shared brain (memory) is the thing that makes it a platform
rather than a collection of apps, and it is the part most worth getting right.

**The ambition is large, and it is worth stating because several decisions only make sense at
that size.** This is meant to become the owner's main tool in daily life, with features added
over years rather than a small fixed set that gets finished. Three things follow. D3, features
never call each other, is the rule that matters most, since it is what keeps growth linear
instead of quadratic. The shell is a real surface rather than a list of links, see Q12. And the
polyglot cost accepted in D14 is paid once per feature rather than once, by one person, which is
a real running cost rather than a one time one.

Related Notion pages: "Northstar OS" (project entry), "Northstar OS, my personal platform",
"Wiki project". v1 material lives under "My Platform (1)" and "Implementing memory" and is
reference only, see "Carried from v1" below.

**Other documents in this folder**

- `open-questions.md` — living list of what is still undecided, and what blocks each
- `Brain/docs/knowledge-model.md` — what the brain is trying to know about a user
- `Brain/docs/event-model.md` — how evidence gets recorded, and the event shape
- `overview.html` — synthesis of the whole design state in one page, with a diagram. Derived
  from the files above rather than authoritative, so regenerate it when decisions change

**A naming convention, since D29 made it matter.** The platform is multi user. Where a rule
says "the owner" it means whoever owns that data, and the rule holds for every account. Where
an example says "Adam" it is an illustration from the first user's case, not a special role.
Those were the same thing until 2026 09 02 and are not any more.

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


It owns two things: memory, and cross feature brokering. It has its own database.

Rationale: if memory lives inside a general backend, that backend slowly absorbs every
feature's data because it is always the path of least resistance. Making the core a client
of the brain rather than its owner keeps that honest.

### D9. We are not designing for standalone extraction


**A feature that uses the brain depends on it hard.** It may still run without it and will not
be much good without it, and nothing is built to soften that.

Whether a feature uses the brain at all is that feature's own business, decided when it is
planned. Some will lean on it heavily, some barely, and one that never touches it is a normal
case rather than an exception. This decision says nothing about which.

**Amended 2026 09 03.** This previously opened "the brain is a hard dependency", flat, which
read as a claim about every feature. It is a claim about the features that use it, and the rest
of the decision never rested on the stronger version.

**What extraction would actually cost.** If a feature ever had to become a standalone product,
you would rebuild the slice of brain behaviour that feature actually used, not the brain. That
is usually a small fraction of it, and saying so keeps this an accepted cost rather than
something that reads like a threat. The earlier wording, "a brain equivalent gets written", made
it sound far larger than it is.

Consequence, and it simplifies a lot: no graceful degradation paths, no fallback adapters,
no speculative portability seams. A feature that uses the brain assumes the brain is there.

### D11. Profile lives in the brain


Identity, goals, values, interests, preferences, and how the owner wants to be worked with.
Explicit user entered facts, and any context read has to merge them with inferred memory.
Splitting them across services means a network join on every read.

Line to hold: the core owns identity, who you are as an account. The brain owns who you
are as a person.

**Corrected 2026 08 04.** An earlier version listed skill levels as profile. They are not.
Profile holds the things the owner is definitionally correct about, so nothing in it moves except
by his own hand. Skill level is a self assessment, which he can be miscalibrated about, so it
lives in the knowledge model and evidence is allowed to move it. See D23.

### D12. Cross feature entity links live in the brain, generically


**Removed 2026 09 03.** It said the brain stores links of the form "this entity in feature A is
that entity in feature B", created by hand through a feature's UI.

**Why it is gone.** A link between two features' entities is not knowledge about a person, it is
a join table, and it ended up in the brain only because the brain was the one place both
features could reach. That is the failure D1 was written against, a general service absorbing
what sits next to it because it is the path of least resistance, applied to the brain instead of
the core. This document already half admitted it, listing the layer as "not memory, but lives in
the brain's database". Its single concrete case, a platform project that is also a wiki page,
also depended on a feature that may never be built.

**What replaces it: nothing, on purpose.** If two features ever need to know they refer to the
same real thing, that is decided then, with a real case in hand rather than a generic mechanism
built for none.

### D14. Language is fixed per layer, and chosen per feature


**Revised 2026 09 02.** This previously read "the stack is three languages" and named C# for
CRUD heavy pieces including possibly the core. Both halves changed: the core is Rust, and
feature languages are no longer pre assigned.

- **Rust** for the core and for infrastructure services.
- **Python** for the brain.
- **TypeScript** for frontends, likely React.
- **Each feature picks its own language when that feature is planned**, not now.

Mobile is deferred entirely, Swift or Expo, decided much later.

**Why the brain is Python.** Its hardest work is consolidation, embeddings, clustering, and
LLM orchestration, and v1's real failure was splitting memory across two languages. One
service in the language where its hardest work lives removes that split by construction.

**Why the core is Rust, and this closes Q7.** Q7 asked Go or C#, and framed it correctly: the
question is not which language is better but which direction each one pushes the design. C#
invites a richer domain model and heavier features. Go invites small boring services.

Rust is Go's direction, more strongly. Building a rich domain model in it is possible but high
friction, and that friction is exactly the behaviour the core is already required to have,
since it must never be down and therefore must hold nothing interesting. The friction is a
forcing function backing a stated goal rather than a tax.

Two consequences, one of them free:

- **Q10 dissolves rather than resolving.** The only argument for putting `projects` inside the
  core codebase was that both would be C#. With the core in Rust the temptation is gone, and
  the availability argument wins by default.
- **The cost is authentication.** C# would have handed over a complete, vetted auth system.
  Rust has excellent primitives and no assembled whole, so the flows are written by hand. That
  is accepted deliberately: it is wanted as practice, the blast radius is one personal
  platform, and the crypto primitives themselves are still library code. See D31.

**Why feature languages are not decided here.** The old list assigned wiki, learning and news
to Python and projects to C# before any of them had been designed. That is a guess wearing the
clothes of a decision. Nothing constrains the choice, and it belongs to the planning phase for
that feature. This previously leaned on D10's tier model, which has since dissolved, see D10.

Note that the number of languages is deliberately not capped. The polyglot cost below is
accepted per service rather than budgeted globally.

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

### D18. First milestone, and it is two independent tracks


**Revised 2026 09 02.** This previously described one milestone, the brain's. There are two,
they do not block each other, and saying so stops either from waiting on the other.

**Core track.** Accounts, authentication, session and token issuance, invite coded
registration, routing, and the feature registry. Finishable, well understood, and the place
the security work lives. See D14, D29 and D31.

**Brain track.** Profile, plus events, plus deliberately dumb retrieval, plus roughly ten
evaluation cases. Nothing else. Small, but it makes everything after it measurable.

**The only seam between them is the user id.** The brain takes it as an opaque parameter from
day one and never resolves it itself. The core issues the token, the brain trusts the verified
claim inside it. That is a ten line stub if the brain starts first, which is why neither track
is blocked.

**Chosen order: core first.** Not because the brain depends on it, since it barely does, but
because the core is small and genuinely finishable, it is the Rust and security practice that
motivated the stack change, and doing it first means the brain gets a real user id from its
first commit instead of a stub to remove later.

**The cost of that order, stated so it is not discovered later.** The original reasoning put
the brain first deliberately: it is where the design risk lives, the core is well understood,
and reaching measurable ground early is what makes the hard part tractable. Core first defers
that. Accepted, because the delay is short at this scale, and because the ten evaluation cases
need no code at all and can be written on paper while the core is in progress. Their seed is
already in `Brain/docs/knowledge-model.md` under "Concrete claims, first pass".

Note added 2026 08 04: D27 lets the owner edit and retire anything in the brain, which makes the
memory centre a general item browser rather than a profile page. That is not first milestone
work, but it is where a meaningful share of the real build sits, so the milestone stays honest
only if the browser is understood as coming later rather than being implied by "profile".

Also 2026 08 04, from D28: there are no features in this milestone, so there are no per feature
context calls to write. It builds the shared retrieval primitives, and **the evaluation harness
is the first caller.** Everything retrievable at this point is manually entered, profile plus
seeded topics plus rules, since consolidation does not exist yet. Events accumulate without
being retrieved, which is correct rather than a gap.

### D29. Multi user from the start, fully isolated


**Decided 2026 09 02.** The platform is built for one person today and will only ever have a
handful of accounts, and it is still built multi user from the first schema.

Reasoning: retrofitting a user scope onto an append only event table and a claim store is a
migration nobody enjoys and everybody postpones. Adding the column before there is any data
costs nothing. This is not an anticipated requirement dressed as a decision, it is the one case
where the cheap option now and the correct option later are the same option.

**Everything in the brain is scoped to a user.** Events, claims, records, profile, procedural
rules, the knowledge model and topic vocabulary. There is no unscoped
memory of any kind.

**Users are fully isolated.** No shared memory, no shared claims, no cross user consolidation.
Consolidation runs over one user's corpus, never the whole table, which is both a correctness
rule and the difference between a clustering bug and a data leak.

**Isolation does not foreclose sharing later, and this is why the isolated version is cheap
rather than limiting.** Sharing, when it comes, belongs inside a feature. From the wiki's point
of view a collaborator is simply another account that logged in, so the wiki can share a page
without the brain ever sharing anything. The isolation boundary sits under the features, not
across them, so feature level sharing is added without reopening this decision.

**Consequences elsewhere:**

- The volume arguments in D19 and D26 stop resting on "single user" and rest on per user scope
  instead. The conclusions are unchanged.
- The authority rules in D22 and D23 are stated as "the owner" rather than "Adam". Same rules,
  no longer written in the first person.
- Every retrieval call carries a verified user identity, see D28.
- A missing scope filter is now a cross user data leak rather than a wrong answer, which is why
  enforcement is a question in its own right rather than a coding convention.

### D30. Notification delivery is its own service


**Decided 2026 09 02.** D13 said reminders go "through the same service" without ever saying
which one. It is a small standalone service, and specifically it is not part of the core.

Why not the core: the core must never be down, which means it must hold nothing that gives a
reason to deploy it. Notification delivery changes when channels, schedules and templates
change, which is often. Putting it in the core is the exact failure D1 warns about, a general
service absorbing things because it is the path of least resistance.

Why not the brain: by D13 it holds no memory, emits nothing to the brain, and reads nothing
from it. It has no reason to live there.

Scope: accept a delivery request from a feature or from the user, hold it until its time or
condition, deliver it, record that it was delivered. Nothing else. Its language is not decided
here, see the open questions.

### D33. Isolation is enforced by the datastore, not by query discipline


**Decided 2026 09 03, closing Q18.** D29 made every row belong to exactly one user. This decides
where that is enforced: in the database, not in the code that queries it.

**The rule is the property, not the mechanism.** Wherever a datastore can refuse to return
another user's rows, it does, and application code is never the only thing standing between two
accounts. For Postgres that mechanism is row level security. A feature on a store that cannot do
this says so and says what it does instead. D10 leaves feature shape open deliberately, but this
is a security property rather than a shape.

**Why, and it is one sentence: the two options fail in opposite directions.** Query layer
discipline fails open, since a forgotten `WHERE` clause returns another person's rows, raises
nothing, and looks like a working feature. Row level security fails closed, since the same
forgotten clause returns zero rows. The class of bug does not get rarer, it stops being able to
leak.

**What it takes to actually hold.** Each of these is a way to have it switched off while looking
configured, which is why they are written down rather than left to setup time:

- The application connects as a role that does not own its tables, since owners bypass policies
  by default. Migrations run as the owner.
- `FORCE ROW LEVEL SECURITY` on every scoped table, so ownership is not a bypass either.
- The user identity reaches the database as a transaction scoped setting, `SET LOCAL`, set
  inside the transaction. Plain `SET` on a pooled connection survives into whatever request
  borrows that connection next, which is a cross user leak caused by the very mechanism adopted
  to prevent them. The footgun does not disappear, it moves from every query site to one place
  in connection handling, which is the whole point.
- Maintenance access is a named, deliberate bypass path rather than an ad hoc superuser
  connection.

**Where it pays beyond the obvious.** D29 calls a missing filter during consolidation "the
difference between a clustering bug and a data leak". Under this decision the consolidation job
sets one user id and cannot see past it, so that case stops being something to test for.

**Accepted cost.** A query returning nothing has a reason that is not visible in the SQL, so
debugging gains a first question. Every connection checkout needs the setting, including
background jobs and anything run by hand. Policies add a predicate to every plan, though
`user_id` belongs in those indexes regardless.

### D34. The shell is not part of the core


**Decided 2026 09 03, splitting Q12.** Q12 asked how ambitious the shell should be and treated
that as one question. It is two. How rich the front page is cannot be settled on paper and stays
open. Where the shell lives and how it gets its data is answerable now, and it binds the core,
which is being built first.

**The core serves the login flow and nothing past it.** Everything before you hold a token has
to be the core's. Everything after it belongs to the shell. `Core/docs/core-architecture.md`
previously listed "serving the shell" among what the core owns, which fails the core's own rule:
a front page changes constantly, and anything that gives you a reason to deploy the core does
not belong in it.

**The shell is its own thing, and most naturally a feature.** Under D10 a feature is whatever
registers itself with the core and talks to the brain through the two contracts. A shell does
exactly that, so nothing new has to be invented to hold it.

**Front page data comes from the browser, not from the brain.** Each feature contributes its own
piece of the page and the browser calls that feature directly with the user's token. The
rejected alternative was the brain brokering it, which D1 permits and which is wrong here: it
turns memory into a dashboard proxy and puts the most visited page in the platform behind a fan
out through a service whose job is knowing things about a person. The browser route also means
adding a feature never means editing something central.

D3 is untouched. No feature calls another. A browser making requests on the user's behalf is the
user, not a feature.

**Accepted cost.** The front page is several parallel calls from the browser rather than one
composed response, every feature has to expose something browser facing for its own piece, and
cross origin authentication has to work for every feature rather than only for navigation. That
is the price of refusing a composition layer in the middle, and a composition layer is exactly
what would have grown into the thing the core is forbidden to be.

---

## Memory layers

Reduced from v1's seven to four:

1. **Profile**, identity, goals, values, interests, preferences and working style. Entered by
   the owner, moved only by the owner. See D23 for what is deliberately not in here
2. **Events**, append only record of what happened
3. **Semantic**, claims and records the brain holds about the user, each linked to whatever it
   rests on. Most are derived from repeated evidence, but not all: the knowledge model's seed
   values, a decision's current status, and reported history are single sourced and live here
   too
4. **Procedural**, versioned rules for how the system should behave

Dropped: working memory (it is workflow state and belongs to whatever runs the workflow),
graph memory (never built in v1), and document memory. Document memory was dropped on the
grounds that the wiki covers long form artifacts, which no longer holds since the wiki may never
exist, but the conclusion stands on its own: if a feature ever needs long form artifacts
remembered, that is raised then. Closed as Q5.

---

## Candidate features

Not a plan, and not a commitment. These are things that might get built, written down so the
platform is designed with them in mind rather than around them. Any of them may be dropped, and
things not on this list will certainly appear. It was Q13 until 2026 09 03, closed because
asking for a complete list of features was never a question that could be answered.

- **Projects.** Likely the first feature after the core.
- **Learning new topics.** The candidate the brain design leans on hardest. D28's worked example
  is a learning context call end to end, the end of session quiz is what replaced passive
  tracking, and Q17 is answered against what learning needs.
- **Wiki.** Uncertain, and deliberately not first. Two other things currently rest on it, see
  below.
- **Books.**
- **An RSVP speed reader.**
- **Controllers and monitors for other devices.**
- **News.** An idea, with no design trace anywhere else in these documents.

**Two things named as features already have homes and are not features.** Reminders is a
delivery capability with its own service, see D13 and D30. A mobile application is a client
surface over features that already exist rather than a feature of its own, see Q14.

**Books, the speed reader, learning and the wiki cluster around reading and learning.** Whether
that is four features or one general one holding four parts is decided when the first of them is
planned. Either shape is identical from the core's and the brain's side, see D10.

**Devices are the only candidate here that stresses anything structural**, and the design
already absorbs it. Telemetry is high volume and almost none of it says anything about a person,
so the feature holds the readings and emits an event only when something meaningful happens.
That is D2 and D20 doing exactly what they were written for, and the filter in
`Brain/docs/knowledge-model.md` is what decides which readings qualify. Worth confirming when
that feature is planned rather than assuming.

**Two things used to rest on the wiki and no longer do.** D12, cross feature entity links, and
Q5, document memory, were both justified by a wiki that may never be built. Both were removed on
2026 09 03 rather than rewritten around a different feature.

---

## Considered and dropped

Recorded so we do not re open them without new information. Items specific to one service are
recorded in that service's folder.

**Designing for standalone extraction.** Explicitly not a priority. See D9.

**The operating system framing.** Northstar OS was a name, not an architecture. Dropped as
a design metaphor.

**Pre assigning a language to every feature.** D14 named Python for wiki, learning and news and
C# for projects before any of them was designed. That is a guess wearing the clothes of a
decision. Feature language is now chosen at feature planning time.

**Splitting memory ownership across two languages.** This was v1's actual failure: the
design said one language owned canonical memory and the other proposed candidates, and the
boundary leaked in practice. Avoided structurally in v2.
