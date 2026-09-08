# The core

Status: exploratory. Nothing is built.

Last updated: 2026 09 08

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
- **The public key** that everything else verifies tokens with, served at an endpoint.

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

`projects` was considered for the core codebase and dropped, since it is simply a feature, and 
features stay out of core, if there are a lot smaller features that don't really deserve it's 
own feature we rather group them in a some kind of general feature later.

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

### The session

**Every service verifies for itself.** The core issues tokens and everything else verifies them
locally, with no call back to the core. Features and the brain read the user identity out of the
token and check the signature themselves. Nothing trusts a user id set by something upstream, and
the obligation that puts on a feature is in `High-level/docs/feature-contract.md`.

The alternative was verifying once at the edge and handing features a user id in a header. It was
dropped for two reasons. Routing is a config file doing path matching, so it cannot check a
signature, which means edge verification would need either a forward auth hook calling the core on
every request or every request routed through the core process, and the second makes the core a
hard dependency for every page load in the platform. That contradicts the tradeoff already
accepted for the nav, where the core being down costs a nav bar and nothing else. The other reason
is that a feature which believes a header holds no authentication logic at all, which is identity
arriving as a parameter wearing a different hat.

**The signing key is asymmetric**, an Ed25519 signed JWT. The core holds the private half and is
the only thing that can mint a token. Everyone else holds the public half and can only check one.
With a shared secret every verifier is also a minter, so any feature could issue a token for any
account, which is a strange property in a platform whose whole shape is features not trusting each
other. The core serves the public key at an endpoint, so nothing hardcodes it or passes it around
by hand, and rotating it stays a core only operation. RS256 is the fallback if a language picked
for some later feature has no Ed25519. PASETO was the alternative and lost on library coverage,
since most of the safety it adds is already bought by never hand writing primitives.

**The token carries a token id and a session id, and nothing reads them.** They are there so a
revocation mechanism can be added later without changing the format. Adding a claim afterwards
means updating every verifier and reissuing every live session; a claim that has always been there
simply starts being read one day.

**The session lives in two cookies.** The access token on `Path=/`, so it reaches the core and
every feature. The refresh token on the path of the refresh endpoint alone, so it never reaches a
feature at all. Both `HttpOnly`, `Secure`, `SameSite=Lax`.

Cookies rather than a token held in JavaScript, because with everything on one origin the browser
attaches the session by itself, so a feature frontend needs no code to be authenticated, and
`HttpOnly` means a script that gets onto a page cannot read the token. A bearer token in
JavaScript is immune to CSRF but is lost outright to any XSS. CSRF is close to fully solvable and
token theft through XSS is not, which is the whole trade.

**CSRF is handled by `SameSite=Lax`, an `Origin` check on anything that changes state, and never
changing state on a GET.** No synchronizer tokens and no double submit, so there is no shared
state between the core and features. Accepting only JSON on mutations closes the same hole a
second time and costs nothing.

**The refresh token is opaque rather than a JWT.** A random string, stored hashed, usable once.
Each refresh mints a new one and marks the old one used. Presenting an already used token means a
copy exists somewhere, so the whole session family is revoked and that person logs in again. This
is also what makes logout mean something and what lets sessions survive a restart, since the truth
is a row rather than a signature.

**The access token lasts thirty minutes, and that number is the revocation window.** A locally
verified token cannot be called back on, so a logout or a stolen laptop stays live until it
expires. The refresh token dies the instant you act, so nothing new can be minted, and the worst
case is one already issued access token surviving half an hour. Fifteen minutes is the textbook
figure and is aimed at a stranger replaying a stolen token, which is not the threat on a platform
with a handful of invited accounts.

**Expiry is not free for features.** An API call gets a 401 and the frontend refreshes and
retries, but a plain page load needs the feature's backend to redirect to the core and back. So a
feature holds signature verification and one redirect, which is little but is not nothing.

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

## What we are not doing

Recorded so it does not get re proposed without new information.

**Letting `nav.js` refresh the session in the background.** It would make token expiry genuinely
invisible to features, since every feature already includes the script. It is not done because the
nav rendering and linking and doing nothing else is what keeps pulling a feature out to deleting
two lines, and an auth helper riding along is exactly the drift that rule exists to prevent. See
the nav in `High-level/docs/platform-architecture.md`.

**Audience scoped tokens, one per feature.** One cookie on the whole domain means every feature
backend receives a credential that would also work against every other feature, so features not
calling each other is a topology rule rather than something cryptography enforces. Per feature
tokens are the real answer and are where the scopes in the registry manifest eventually lead. Not
built now, because it is a token exchange mechanism to design and every feature is the owner's own
code.

**A revocation feed, so a session can be killed instantly.** Every version of this works the same
way: the verifier has to learn something the token cannot tell it, so it is either a lookup on
every request or a list the core publishes and each service caches. The cheapest shape is a
session epoch per account, one integer, bumped to invalidate everything that account holds. It is
not built because thirty minutes is an acceptable window here, and because the token already
carries the identifiers that would make adding it later a small piece of work rather than a
migration. Whatever is built has to serve stale on a core outage, since failing closed would mean
the core going down logs the whole platform out.

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
