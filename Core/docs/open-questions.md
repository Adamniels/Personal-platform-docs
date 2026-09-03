# Open questions, core

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

Questions answered inside the core. Questions whose answer needs the brain as well live in
`High-level/docs/open-questions.md`.

---

## Open

### Q8. What the feature manifest contains


How a feature registers itself: routes, read API version, entity types it exposes, scopes
it requests. Needed before the second feature exists, not before the first.

### Q9. Routing implementation


Path based routing to different processes may be a Caddy or Traefik config file rather than
code, which would leave only a registry and token verification as an actual service.

---

## Resolved

### Q7. Core in Go or C#


**Resolved 2026 09 02: neither. The core is Rust.** See D14.

The question was framed correctly, which is why it survived a language it did not list. It
asked not which language is better but which direction each pushes the design: C# invites a
richer domain model and heavier features, Go invites small boring services.

Rust is Go's direction, more strongly. A rich domain model in Rust is possible but high
friction, and the core is already required to be small and boring because it must never be
down. The friction backs a stated goal instead of fighting one, which is the strongest form
this argument can take.

Two things came with it. Q10 dissolved, since the only reason to consider putting `projects`
in the core was that both would be C#. And authentication became work rather than a
dependency, which is accepted deliberately in D31 as practice with a blast radius of one
personal platform.

One correction to this question's own wording, worth keeping because the mistake is easy to
repeat: it said Go "pushes complexity into the brain instead". It does not. Under D2 and D7
complexity displaced from a thin core lands in *features*, since the brain is forbidden from
deciding anything in a feature's domain. That made the Go and Rust direction more aligned with
the rest of the design than the sentence suggested.

### Q10. Does projects live in the core codebase


**Dissolved 2026 09 02 rather than answered.** The only argument in favour was that the core
and `projects` would both be C#. With the core in Rust under D14 there is nothing left on that
side, and the argument against stands unopposed: the core does auth and routing and must never
be down, and a bug in a scrum board should not take out routing for everything.

D14 also stopped pre assigning languages to features, so what `projects` is written in is now
decided when `projects` is planned.
