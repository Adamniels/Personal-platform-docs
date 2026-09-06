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
- **Routing** to features, which is a reverse proxy. Path based routing to separate processes is
  most likely a Caddy or Traefik config file rather than code. It also puts everything behind one
  domain, which is what makes cookies and requests between the core and features simple.
- **The feature registry**, where a feature declares itself. A manifest would carry its routes,
  its read API version, the entity types it exposes and the scopes it requests. What exactly
  goes in it matters once there is a second feature to register, since one feature does not need
  a registry to find itself.
- **Its own frontend**, meaning the login pages, the front page, and shared navigation.
- **Composing the front page**, meaning calling the display endpoints of whichever features you
  chose and assembling the result.

## What the core grows into, and what stays out

The core changes as features arrive, and that is the workflow rather than a smell. Adding a
feature means three things outside the feature itself: a proxy entry, connecting it to the brain,
and deciding whether anything of it belongs on the front page. There is no rule against the core
changing, and there is no requirement that it never be down. Sessions survive a restart, since
tokens are verified locally rather than by calling back.

Two things still do not belong in it, for their own reasons rather than because a test says so.

**Domain data.** A general backend slowly absorbs every feature's data because it is always the
path of least resistance. Features own their data, the brain owns what it means, and the core
holds neither. Note that composing the front page does not breach this: a feature sends display
data for that moment, the core renders it and keeps nothing.

**Notification delivery.** It needs persistence and background scheduling, which is a different
runtime shape from a service that answers requests. Its own small service, see
`High-level/docs/platform-architecture.md`.

`projects` was considered for the core codebase and dropped, since a bug in a scrum board should
not take out routing for everything. Its only argument was that both would have been C#, which
went away with Rust.

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

## What a feature exposes for the front page

Optional, per feature, and not a contract. When you build a feature you decide whether anything
of it belongs on the front page. If it does, it exposes an endpoint returning display data, and
the core calls it with your token forwarded.

It returns data, not rendered output. The core decides how it looks, which is what keeps the page
coherent rather than a patchwork, and what keeps the endpoint useful to anything else that might
want to read it later.

Nothing is required. A feature with nothing worth showing exposes nothing, and the registry
records which features have one and where.

---

## What shapes the core from elsewhere

All in `High-level/docs/platform-architecture.md`:

- The core is Rust, and why.
- The core track goes first, and what is in it.
- Multi user from the start, and isolation enforced by the datastore rather than by query
  discipline. That binds the accounts schema and how the core connects to it.
- Notification delivery is its own service, not this one.
- Profile lives in the brain, which fixes the account versus person line.

And in `High-level/docs/feature-contract.md`: both directions of the two contracts carry a
verified user identity, which the core is what issues.
