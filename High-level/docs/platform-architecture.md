# Platform architecture

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

This is the top of the documentation. It holds what you need to understand the system before
building any part of it: what the platform is, what services exist, and the decisions that
constrain more than one of them.

Anything you only need when you sit down to write a particular service lives in that service's
folder instead, `Brain/docs/` or `Core/docs/`. Those documents point back here. This one points
down to them for detail rather than repeating it.

**On numbering.** Decision numbers (D1 to D32) and question numbers (Q1 to Q19) are global
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


It owns three things: memory, cross feature brokering, and cross feature entity links.
It has its own database.

Rationale: if memory lives inside a general backend, that backend slowly absorbs every
feature's data because it is always the path of least resistance. Making the core a client
of the brain rather than its owner keeps that honest.

### D9. We are not designing for standalone extraction


The brain is a hard dependency. A feature may run without it but will not be useful without
it. If a feature ever needs to become a standalone product, a brain equivalent gets written
at that point.

Consequence, and it simplifies a lot: no graceful degradation paths, no fallback adapters,
no speculative portability seams. Features assume the brain is there.

comment to remove after we gone through this: not all features need to have a hard dependency on the brain,
that is not the point, just in the cases where a feature use the brain it should/will not work as good, meaning
we have to implement something like the brain just for the feature if we want it as a standalone project.

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


Stored as: this entity in feature A relates to that entity in feature B. Created by hand,
by the owner, through a feature's UI which asks the brain what could be linked rather than
asking the other feature.

The only concrete case today is a platform project that also exists in the wiki. The
generic form is chosen not because reuse is expected but because the alternative is a
wiki id column on the project table, which breaks D3. Consistency, not anticipated reuse.

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
rules, the knowledge model, topic vocabulary and cross feature links. There is no unscoped
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

---

## Memory layers

Reduced from v1's seven to four, plus links:

1. **Profile**, identity, goals, values, interests, preferences and working style. Entered by
   the owner, moved only by the owner. See D23 for what is deliberately not in here
2. **Events**, append only record of what happened
3. **Semantic**, claims and records the brain holds about the user, each linked to whatever it
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
