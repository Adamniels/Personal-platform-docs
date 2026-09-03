# The core

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

The platform's infrastructure: the thing every feature is reached through, and where nothing
interesting lives. The platform wide thinking that places it is in
`High-level/docs/platform-architecture.md`.

This is the least specified part of the project, and that reflects how much has actually been
worked out rather than a gap in the writing. Worth knowing, given the core goes first.

---

## What the core owns

- **Authentication**, and the account record.
- **Identity at the account level.** The core owns who you are as an account, the brain owns who
  you are as a person.
- **Routing** to features. Path based routing to separate processes may end up being a proxy
  config file, Caddy or Traefik, rather than code, which would leave the core as a registry and
  token verification and not much else. Worked out when it is built.
- **The feature registry**, where a feature declares itself. A manifest would carry its routes,
  its read API version, the entity types it exposes and the scopes it requests. What exactly
  goes in it matters once there is a second feature to register, since one feature does not need
  a registry to find itself.
- **Serving the login flow**, meaning the pages you see before you hold a token. Everything
  after that belongs to the shell, which is not the core.

## What the core must never hold

The test: if it would ever give you a reason to deploy the core, it does not belong in the core.

The core must never be down, which is the whole reason it is kept boring. Authentication passes
the test, since it changes rarely. Three things have already failed it:

- **Domain data of any kind.** A general backend slowly absorbs every feature's data because it
  is always the path of least resistance, and making the core a client of the brain rather than
  its owner is what keeps that honest.
- **Notification delivery**, which changes whenever channels, schedules or templates change.
- **The shell**, which is a front page and changes constantly.

`projects` was considered for the core codebase and dropped. Its only argument was that both
would have been C#, which went away with Rust, and the argument against stands unopposed: a bug
in a scrum board should not take out routing for everything.

Rust helps here rather than relying on discipline. Building a rich domain model in it is high
friction, so the language makes the core boring by default.

---

## Authentication

The core implements its own rather than adopting an identity provider or an off the shelf
framework. This is chosen with the tradeoff understood and it is not the general recommendation.
The usual advice against writing your own auth exists for multi tenant systems holding other
people's data at scale. Here the blast radius is a personal platform with a handful of invited
accounts, and the learning is a stated goal of the project rather than a side effect.

**The line while doing it: build the flows, never the primitives.** Registration, login,
sessions, token issuance and verification, refresh rotation, logout and password reset are
written by hand. Hashing, signing and randomness are library calls, always. Hand written crypto
is where hand written auth actually gets broken, and that part is firm.

**Registration is open but invite coded.** A single use code generated from a terminal command,
consumed by exactly one registration, and expiring. That keeps signup a real flow worth building
rather than a hardcoded bootstrap account, without leaving the platform open.

**An escape hatch, kept deliberately.** Token issuance and token verification stay behind
separate interfaces, and everything downstream only ever verifies, so an identity provider can
be dropped in later without touching the brain or any feature.

---

## What shapes the core from elsewhere

All in `High-level/docs/platform-architecture.md`:

- The core is Rust, and why.
- The core track goes first, and what is in it.
- Multi user from the start, and isolation enforced by the datastore rather than by query
  discipline. That binds the accounts schema and how the core connects to it.
- Notification delivery and the shell are their own things, not this one.
- Profile lives in the brain, which fixes the account versus person line.

And in `High-level/docs/feature-contract.md`: both directions of the two contracts carry a
verified user identity, which the core is what issues.
