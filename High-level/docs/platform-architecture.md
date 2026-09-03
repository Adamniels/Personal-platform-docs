# Platform architecture

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

The top of the documentation. What the platform is, what services exist, and the thinking that
shapes more than one of them. Anything you only need when you sit down to write a particular
service lives in that service's folder instead, `Brain/docs/` or `Core/docs/`.

These are notes, not rulings. Almost everything here is a current position with the reasoning
attached, so it can be argued with. A few things are written as rules, and where that happens it
says what breaks without them.

**Read next**

- `High-level/docs/feature-contract.md` — what a feature is, and what it may depend on
- `Core/docs/` — the core in depth
- `Brain/docs/` — the brain in depth, including what it is trying to know and how evidence
  gets in

**A naming convention.** The platform is multi user. Where something says "the owner" it means
whoever owns that data, and it holds for every account. Where an example says "Adam" it is an
illustration from the first user's case, not a special role.

Related Notion pages: "Northstar OS" (project entry), "Northstar OS, my personal platform",
"Wiki project". v1 material lives under "My Platform (1)" and "Implementing memory" and is
reference only, see "Carried from v1" at the bottom of `Brain/docs/brain-architecture.md`.

---

## What this is

The second version of a personal platform. A stable core plus features built one at a time,
independently. A shared brain, meaning memory, is what makes it a platform rather than a
collection of apps, and it is the part most worth getting right.

The ambition is large, and it is worth saying because a few things only make sense at that
size. This is meant to become the owner's main tool in daily life, with features added over
years rather than a small fixed set that gets finished. Three things follow from that. Features
not calling each other is what keeps growth linear instead of quadratic, which matters more the
more features there are. The shell is a real surface rather than a list of links. And the cost
of running several languages is paid once per feature rather than once, by one person, so it is
a running cost rather than a one time one.

## The framing

Two questions that kept getting bundled together, and are worth holding apart:

1. **Where does a feature's UI and data live?** Isolated. Own frontend, own backend, own
   database.
2. **What is a feature allowed to depend on?** A small fixed set of platform contracts, and
   not much else.

Isolation of implementation, coupling through contracts. Most of what follows comes out of
keeping those two separate.

---

## The brain is its own service

It owns two things, memory and cross feature brokering, and it has its own database.

The reason it is separate rather than part of a general backend: if memory lives inside a
general backend, that backend slowly absorbs every feature's data, because it is always the
path of least resistance. Making the core a client of the brain rather than its owner is what
keeps that honest.

### The brain as a dependency

Features use the brain as much or as little as they want. Most will lean on it, some barely,
and one that never touches it is fine. This says nothing about which.

For the ones that do use it, nothing is built to soften the dependency. No graceful degradation
paths, no fallback adapters, no speculative portability seams. A feature that uses the brain
assumes the brain is there.

Extraction is not something to design for. If a feature ever had to become a standalone
product, you would rebuild whatever slice of brain behaviour that feature actually used, and
how much work that is depends entirely on how deep it went. That cost is accepted rather than
engineered around.

### Profile lives in the brain

Identity, goals, values, interests, preferences, and how the owner wants to be worked with.
All of it explicitly entered, and any context read has to merge it with inferred memory, so
splitting it across services would mean a network join on every read.

The line to hold: the core owns who you are as an account, the brain owns who you are as a
person.

Skill level is deliberately not in here. Profile holds the things the owner is definitionally
right about, so nothing in it moves except by his own hand. A self assessment is something he
can be miscalibrated about, so level sits in the knowledge model where evidence is allowed to
move it. `Brain/docs/trust-model.md` has the full picture of which store holds what and who can
change it.

### No model calls on the read path

Reads are SQL plus vector, deterministic, tens of milliseconds. All the expensive reasoning
happens in consolidation, offline. That is what makes coupling every feature to one shared
brain safe on latency grounds.

A feature fetches a context packet once per run, not per page render. Writes are events, fire
and forget, never blocking a user action.

Caching the profile snapshot is fine but has to invalidate on write, which is trivial at this
scale. A directly written fact has to affect the very next request, so a stale cache quietly
breaks that.

---

## Languages

- **Rust** for the core and for infrastructure services.
- **Python** for the brain.
- **TypeScript** for frontends, probably React.
- **Each feature picks its own** when that feature gets planned.

The number of languages is deliberately not capped.

**Why the brain is Python.** Its hardest work is consolidation, embeddings, clustering and
model orchestration. v1's real failure was splitting memory across two languages, and one
service in the language where its hardest work lives removes that split by construction.

**Why the core is Rust.** Less about the language than the direction it pushes. C# invites a
richer domain model and heavier features; Go invites small boring services; Rust is Go's
direction, more strongly. Building a rich domain model in Rust is possible but high friction,
and the core is supposed to hold nothing interesting anyway, so the friction backs the goal
instead of fighting it.

One thing that is easy to get backwards: a thin core does not push complexity into the brain.
The brain is not allowed to decide anything in a feature's domain, so complexity displaced from
the core lands in *features*.

Two things came with the Rust choice. Putting `projects` inside the core codebase stopped being
tempting, since its only argument was that both would have been C#. And authentication became
work rather than something adopted, since Rust has excellent primitives and no assembled auth
system. That is wanted as practice, and `Core/docs/core-architecture.md` covers how it gets
built and where the line sits.

**Why feature languages are not picked here.** An earlier list assigned wiki, learning and news
to Python and projects to C# before any of them had been designed, which is a guess wearing the
clothes of a decision. Nothing constrains the choice, so it belongs to the planning phase for
that feature.

**Mobile** is deferred entirely. Swift or Expo, much later, and partly determined by whether the
web frontends end up sharing enough to be worth reusing.

**What running several languages costs**, accepted knowingly: separate builds, separate tooling,
no shared types, and contract drift caught at runtime. The mitigation is schema first contracts,
OpenAPI or protobuf, with generated clients on both sides, and it is worth committing to early
rather than retrofitting.

---

## Multi user, and how isolation is enforced

The platform is built for one person today and will only ever have a handful of accounts. It is
still multi user from the first schema, because retrofitting a user scope onto an append only
event table and a claim store is a migration nobody enjoys and everybody postpones. Adding the
column before there is any data costs nothing. This is the one case where the cheap option now
and the correct option later are the same option.

**Everything in the brain is scoped to a user.** Events, claims, records, profile, procedural
rules, the knowledge model and topic vocabulary. There is no unscoped memory of any kind, and
no unscoped call in either direction of the feature contract. Consolidation runs over one
user's corpus, never the whole table.

Those two are written as rules rather than intentions, and here is what goes wrong without
them: a missing scope filter is not a wrong answer, it is one person's data appearing in
another person's account, and during consolidation it is the difference between a clustering
bug and one user's evidence being written into someone else's memory.

**Isolation is enforced by the datastore, not by query discipline.** Wherever a datastore can
refuse to return another user's rows, it does, and application code is never the only thing
standing between two accounts. For Postgres that means row level security. A feature on a store
that cannot do this says so and says what it does instead.

The reason is that the two approaches fail in opposite directions. Query discipline fails open:
a forgotten `WHERE` clause returns another person's rows, raises nothing, and looks like a
working feature. Row level security fails closed: the same forgotten clause returns zero rows.
The class of bug does not get rarer, it stops being able to leak.

Four things have to hold for that to be real, and each one is a way to have it switched off
while looking configured:

- The application connects as a role that does not own its tables, since owners bypass policies
  by default. Migrations run as the owner.
- `FORCE ROW LEVEL SECURITY` on every scoped table, so ownership is not a bypass either.
- The user identity reaches the database as a transaction scoped setting, `SET LOCAL`, set
  inside the transaction. Plain `SET` on a pooled connection survives into whatever request
  borrows that connection next, which is a cross user leak caused by the mechanism adopted to
  prevent them. The footgun does not disappear, it moves from every query site to one place in
  connection handling, which is the point.
- Maintenance access is a named, deliberate bypass path rather than an ad hoc superuser
  connection.

What it costs: a query returning nothing has a reason that is not visible in the SQL, so
debugging gains a first question. Every connection checkout needs the setting, including
background jobs and anything run by hand. Policies add a predicate to every plan, though
`user_id` belongs in those indexes regardless.

**Isolation does not foreclose sharing later**, which is why the isolated version is cheap
rather than limiting. Sharing, if it ever comes, belongs inside a feature. From the wiki's point
of view a collaborator is simply another account that logged in, so the wiki can share a page
without the brain sharing anything. The boundary sits under the features rather than across
them.

---

## The shell

The shell is the front door and the corridor: what you see after logging in, and how you get
from one feature to another. It is not optional, because you have to land somewhere.

**It is not part of the core.** The core serves the login flow, everything before you hold a
token. Everything after that belongs to the shell. A front page changes constantly, and
anything that gives you a reason to deploy the core does not belong in the core.

**It is most naturally just a feature.** A feature is whatever registers itself with the core
and talks to the brain, and a shell does exactly that, so nothing new has to exist to hold it.

**Front page data comes from the browser, not from the brain.** Each feature contributes its own
piece of the page and the browser calls that feature directly with the user's token. The
alternative was the brain brokering it, which it could, but that turns memory into a dashboard
proxy and puts the most visited page in the platform behind a fan out through a service whose
job is knowing things about a person. Going through the browser also means adding a feature
never means editing something central. No feature calls another either way: a browser making
requests on the user's behalf is the user.

What that costs: the front page is several parallel browser calls rather than one composed
response, every feature has to expose something browser facing for its own piece, and cross
origin authentication has to work everywhere rather than only for navigation. That is the price
of refusing a composition layer in the middle, and a composition layer is what would have grown
into the thing the core is supposed not to be.

**How rich it should be is genuinely open.** A list of links, a front page with real content on
it, or one unified product where every feature shares a design system and navigation feels
seamless. That is answered by using the thing rather than by reasoning about it. Two things
worth carrying into it: the ambition of a main daily tool makes the list of links unlikely, and
the unified product is the largest permanent cost in the whole plan, larger than running several
languages, because every feature ever built has to be dragged into the same design system
forever.

---

## Notifications and reminders

Delivering something to the user at a time or under a condition is its own small service. Not
the core and not the brain.

Not the core, because notification delivery changes whenever channels, schedules or templates
change, which is often, and the core must stay boring. Putting it there is the exact drift the
brain exists to avoid, a general service absorbing things because it is the path of least
resistance.

Not the brain, because it holds no memory, writes nothing to the brain and reads nothing from
it. `High-level/docs/feature-contract.md` covers why a reminder is never treated as evidence
about the person.

Scope: accept a delivery request from a feature or from the user, hold it until its time or
condition, deliver it, record that it was delivered. Nothing else.

Its language gets picked when it is designed. It is scheduling and delivery, nothing model
shaped, and it holds no memory, so Rust would fit and would extend the same practice motive
behind the core.

---

## The first milestone

Two tracks. They do not block each other, and saying so stops either from waiting on the other.

**Core track.** Accounts, authentication, session and token issuance, invite coded
registration, routing, and the feature registry. Finishable, well understood, and where the
security work lives.

**Brain track.** Profile, plus events, plus deliberately dumb retrieval, plus roughly ten
evaluation cases. Nothing else. Small, but it makes everything after it measurable.

**The only seam between them is the user id.** The brain takes it as an opaque parameter from
day one and never resolves it itself. The core issues the token, the brain trusts the verified
claim inside it. That is a ten line stub if the brain starts first, which is why neither track
is blocked.

**Core first**, not because the brain depends on it, since it barely does, but because the core
is small and genuinely finishable, it is the Rust and security practice that motivated the stack
in the first place, and doing it first means the brain gets a real user id from its first commit
instead of a stub to remove later.

The cost of that order, worth stating so it is not discovered later: the original reasoning put
the brain first deliberately, since that is where the design risk lives and reaching measurable
ground early is what makes the hard part tractable. Core first defers that. Acceptable, because
the delay is short at this scale and because the ten evaluation cases need no code at all and
can be written on paper while the core is in progress. Their seed is in
`Brain/docs/knowledge-model.md` under "Concrete claims, first pass".

Two things this milestone does not include, and both are easy to assume it does. The memory
centre is a general item browser rather than a profile page, and a meaningful share of the real
build sits there, but it comes later. And there are no features yet, so there are no per feature
context calls to write: this builds the shared retrieval primitives, and the evaluation harness
is their first caller. Everything retrievable at that point is manually entered, since
consolidation does not exist yet, so events accumulate without being retrieved. That is correct
rather than a gap.

---

## Memory layers

Reduced from v1's seven to four:

1. **Profile**, identity, goals, values, interests, preferences and working style. Entered by
   the owner, moved only by the owner.
2. **Events**, append only record of what happened.
3. **Semantic**, claims and records the brain holds about the user, each linked to whatever it
   rests on. Most are derived from repeated evidence, but not all: the knowledge model's seed
   values, a decision's current status, and reported history are single sourced and live here
   too.
4. **Procedural**, versioned rules for how the system should behave.

Dropped: working memory, since it is workflow state and belongs to whatever runs the workflow;
graph memory, never built in v1; and document memory. Document memory was originally dropped on
the grounds that the wiki covers long form artifacts, and the wiki may never be built, but the
conclusion stands on its own. If a feature ever needs long form artifacts remembered, that gets
worked out then.

---

## Candidate features

Not a plan, and not a commitment. Things that might get built, written down so the platform is
designed with them in mind rather than around them. Any of them may be dropped, and things not
on this list will certainly appear. There is no point trying to write the complete list.

- **Projects.** Likely the first feature after the core.
- **Learning new topics.** The one the brain design leans on hardest. The worked example of a
  per feature context call is a learning call end to end, the end of session quiz is what
  replaced passive tracking, and how the knowledge model represents level gets answered against
  what learning needs.
- **Wiki.** Uncertain, and deliberately not first.
- **Books.**
- **An RSVP speed reader.**
- **Controllers and monitors for other devices.**
- **News.** An idea, with no design behind it yet.

**Two things named as features are not features.** Reminders is a delivery capability with its
own service, above. A mobile application is a client surface over features that already exist,
and mobile itself is deferred.

**Books, the speed reader, learning and the wiki cluster around reading and learning.** Whether
that is four features or one general one holding four parts gets decided when the first of them
is planned. Either shape looks identical from the core's and the brain's side.

**Devices is the only candidate that stresses anything structural**, and the design already
absorbs it. Telemetry is high volume and almost none of it says anything about a person, so the
feature keeps the readings and emits an event only when something meaningful happens. The filter
in `Brain/docs/knowledge-model.md` is what decides which readings qualify. Worth confirming when
that feature is planned rather than assuming.

---

## What we are not doing

Recorded so it does not get re proposed without new information. Things specific to one service
are in that service's folder.

**Designing for standalone extraction.** Covered above. The cost is accepted rather than
engineered around.

**Cross feature entity links in the brain.** The idea was that the brain would hold links of the
form "this thing in feature A is that thing in feature B", created by hand through a feature's
UI. A link like that is a join table rather than anything the brain knows about a person, and it
would live in the brain only because the brain is the one place two features can both reach,
which is the drift the brain exists to avoid. Its one concrete case was a project that is also a
wiki page, which depended on a feature that may never exist. If two features ever need to know
they point at the same real thing, that gets worked out then with a real case in hand.

**Sorting features into tiers.** There was a model with three of them, federated, embedded and
linked. Two were empty and the third was every feature, so it sorted nothing. Embedded meant a
feature living inside the brain's process and database, which is a written invitation to the
drift the brain exists to avoid. A feature is whatever registers with the core and talks to the
brain, and what sits inside it is its own business.

**The operating system framing.** Northstar OS was a name, not an architecture. Dropped as a
design metaphor.

**Pre assigning a language to every feature.** Covered under languages.

**Splitting memory ownership across two languages.** This was v1's actual failure: the design
said one language owned canonical memory and the other proposed candidates, and the boundary
leaked in practice. Avoided structurally in v2.
