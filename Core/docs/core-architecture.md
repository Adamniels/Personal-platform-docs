# The core

Status: exploratory. Nothing is built.

Last updated: 2026 09 02

The core is the platform's infrastructure: the thing every feature is reached through and
nothing interesting lives inside. This document is what the core is and what it owns. The
decisions that place it inside the platform are in `High-level/docs/platform-architecture.md`.

**This is the least specified part of the project.** Of thirty one decisions the core owns one,
plus a boundary line inside D11. That is not an oversight in the documentation, it reflects how
much has actually been decided, and it is worth knowing given D18 puts the core track first.

---

## What the core owns

- **Authentication**, and the account record. See D31.
- **Identity at the account level.** D11 draws the line: the core owns who you are as an
  account, the brain owns who you are as a person.
- **Routing** to features. How much of this is code at all is open, see Q9.
- **The feature registry**, where a feature declares itself. Contents open, see Q8.
- **Serving the shell.** What the shell actually is, is open, see Q12.

## What the core must never hold

**The rule: if it would ever give you a reason to deploy the core, it does not belong in the
core.**

The core must never be down, which is the whole reason it is kept boring. Authentication passes
that test, since it changes rarely. Two things have already been ruled out by it:

- **Domain data of any kind.** D1 states the failure directly: a general backend slowly absorbs
  every feature's data because it is always the path of least resistance. Making the core a
  client of the brain rather than its owner is what keeps that honest.
- **Notification delivery**, which changes whenever channels, schedules or templates change.
  It is its own service, see D30.

`projects` was considered for the core codebase and rejected, see Q10 in the questions file.

D14 chose Rust partly for this reason: building a rich domain model in it is high friction, so
the language makes the core boring rather than relying on discipline to keep it that way.

---

## Decisions

### D31. Authentication is built by hand, and registration is invite coded


**Decided 2026 09 02.** The core implements its own authentication rather than adopting an
identity provider or an off the shelf framework.

This is chosen with the tradeoff understood and it is not the general recommendation. The usual
advice against writing your own auth exists for multi tenant systems holding other people's
data at scale. Here the blast radius is a personal platform with a handful of invited accounts,
and the learning is a stated goal of the project rather than a side effect.

**The line held while doing it: build the flows, never the primitives.** Registration, login,
sessions, token issuance and verification, refresh rotation, logout and password reset are
written. Hashing, signing and randomness are library calls, always. Hand written crypto is
where hand written auth actually gets broken.

**Registration is open but invite coded.** A single use code is generated from a terminal
command, is consumed by exactly one registration, and expires. That keeps signup a real flow
worth building rather than a hardcoded bootstrap account, without leaving the platform open.

**Escape hatch, kept deliberately.** Token issuance and token verification stay behind separate
interfaces. Everything downstream only ever verifies, so an identity provider can be dropped in
later without touching the brain or any feature.

---

## Also constrained by

Decisions that live in other folders and bind the core anyway.

- **D11**, profile lives in the brain. Fixes the account versus person line.
- **D14**, language is fixed per layer. The core is Rust.
- **D18**, first milestone. The core track goes first, and its scope is listed there.
- **D29**, multi user from the start. Accounts, tokens and every downstream scope.
- **D30**, notification delivery is its own service, and specifically not this one.
- **The two contracts**, in `High-level/docs/feature-contract.md`. Both directions carry a
  verified user identity, which the core issues.
