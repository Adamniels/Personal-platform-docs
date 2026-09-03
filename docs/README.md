# Documentation index

Status: exploratory. Nothing is built.

Last updated: 2026 09 03

These are planning notes, not specifications. Everything here is a current position with the
reasoning attached, and almost all of it can be argued with. A few things are written as rules,
and where that happens the document says what breaks without them.

Planning is split by **level**, not by subsystem. The test for where something belongs:

> Who is constrained by it? If it constrains features or the platform, it is high level. If you
> only need it when you sit down to write that service, it belongs to that service.

**Nothing is written in two places.** There are no summaries or restatements, because a copy
goes stale and then quietly lies. Documents point at each other instead, in both directions.

Each document describes the current state of its area, including the parts deliberately left
open and what would settle them. There is no separate list of loose ends.

---

## Where to start

1. `High-level/docs/platform-architecture.md` — what the platform is and how its parts relate
2. `High-level/docs/feature-contract.md` — what a feature is and what it may depend on
3. `Core/docs/` or `Brain/docs/` — whichever you are about to build

## Layout

| Folder             | Holds                                                                       |
| ------------------ | --------------------------------------------------------------------------- |
| `High-level/docs/` | The platform, the services that exist, and anything shaping more than one   |
| `Core/docs/`       | The core: auth, accounts, routing, the registry                             |
| `Brain/docs/`      | The brain: memory, the trust model, events, retrieval                       |
| `docs/`            | This index, and anything that fits nowhere else                             |

## The files

**High level**

- `platform-architecture.md` — the brain as a service, dependency and extraction, languages,
  multi user and isolation, the shell, notifications, the first milestone, memory layers,
  candidate features, and what is deliberately not being done
- `feature-contract.md` — what a feature is, what it owns, what it may depend on, the two
  contracts, and where reminders sit

**Core**

- `core-architecture.md` — what the core owns, what it must never hold, and how authentication
  gets built

**Brain**

- `brain-architecture.md` — per feature modules, retrieval, evaluation, consolidation, and the
  brain's own surface
- `trust-model.md` — which store holds what, who can change it, scope, confidence and currency,
  editing, and the review queue
- `knowledge-model.md` — what the brain is trying to know about a person, and the filter for
  what gets in at all
- `event-model.md` — the event shape, verbs, relations, and what consolidation does
