# Documentation index

Status: exploratory. Nothing is built.

Last updated: 2026 09 02

Planning is split by **level**, not by subsystem. The test for where something belongs:

> Who is constrained by it? If it constrains features or the platform, it is high level. If you
> only need it when you sit down to write that service, it belongs to that service.

And for questions:

> Does answering it require knowing about more than one area, or does it merely affect several?
> Requires more than one, it is a high level question. Merely affects, it lives where it gets
> answered.

**Nothing is written in two places.** There are no summaries or restatements, because a copy
goes stale and then quietly lies. Documents point at each other instead, in both directions.

---

## Where to start

1. `High-level/docs/platform-architecture.md` — what the platform is and how its parts relate
2. `High-level/docs/feature-contract.md` — what a feature is and what it may depend on
3. `Core/docs/` or `Brain/docs/` — whichever you are about to build
4. The open questions file in each folder — what is left to solve there

## Layout

| Folder             | Holds                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------- |
| `High-level/docs/` | The platform, the services that exist, and every decision constraining more than one of them |
| `Core/docs/`       | The core: auth, accounts, routing, the registry                                              |
| `Brain/docs/`      | The brain: memory, the trust model, events, retrieval                                        |
| `docs/`            | This index, and anything that fits nowhere else                                              |

## Numbering

Decision and question numbers are **global and permanent**. They are never renumbered when a
document moves, because decisions cite each other and several carry amendment history. This
index is how you find one.

## Decisions

| #   | Title                                                                            | Lives in                                   |
| --- | -------------------------------------------------------------------------------- | ------------------------------------------ |
| D1  | The brain is its own service                                                     | `High-level/docs/platform-architecture.md` |
| D2  | Features own their data. The brain owns what that data means about you           | `High-level/docs/feature-contract.md`      |
| D3  | Features never call each other                                                   | `High-level/docs/feature-contract.md`      |
| D4  | The brain does not connect to feature databases. Features expose read APIs       | `High-level/docs/feature-contract.md`      |
| D5  | Per feature code in the brain, written as needed                                 | `Brain/docs/brain-architecture.md`         |
| D6  | Unified interface facing out, separate modules facing in                         | `Brain/docs/brain-architecture.md`         |
| D7  | The brain never makes a decision that belongs to a feature's domain              | `High-level/docs/feature-contract.md`      |
| D8  | Each feature wraps brain access behind one internal port                         | `High-level/docs/feature-contract.md`      |
| D9  | We are not designing for standalone extraction                                   | `High-level/docs/platform-architecture.md` |
| D10 | Feature tiers                                                                    | `High-level/docs/feature-contract.md`      |
| D11 | Profile lives in the brain                                                       | `High-level/docs/platform-architecture.md` |
| D12 | Cross feature entity links live in the brain, generically                        | `High-level/docs/platform-architecture.md` |
| D13 | Reminders and notifications are a delivery capability, not a feature             | `High-level/docs/feature-contract.md`      |
| D14 | Language is fixed per layer, and chosen per feature                              | `High-level/docs/platform-architecture.md` |
| D15 | No LLM calls on the read path                                                    | `High-level/docs/platform-architecture.md` |
| D16 | Retrieval evaluation exists from early on                                        | `Brain/docs/brain-architecture.md`         |
| D17 | Consolidation is idempotent and re runnable from the start                       | `Brain/docs/brain-architecture.md`         |
| D18 | First milestone, and it is two independent tracks                                | `High-level/docs/platform-architecture.md` |
| D19 | One table for all events, with an extensible schema                              | `Brain/docs/event-model.md`                |
| D20 | Observation granularity is a per feature decision                                | `High-level/docs/feature-contract.md`      |
| D21 | Records apply immediately. Claims wait for consolidation                         | `High-level/docs/feature-contract.md`      |
| D22 | Nothing is destroyed, and only the owner retires                                 | `Brain/docs/trust-model.md`                |
| D23 | Authority follows the store, not the source                                      | `Brain/docs/trust-model.md`                |
| D24 | Scope is topic and entity. Never feature                                         | `Brain/docs/trust-model.md`                |
| D25 | Standing rules are created deliberately. Conversation instructions are temporary | `Brain/docs/trust-model.md`                |
| D26 | Confidence, currency and status are three different things                       | `Brain/docs/trust-model.md`                |
| D27 | Editing changes provenance, and every edit records a judgment                    | `Brain/docs/trust-model.md`                |
| D28 | Retrieval is per feature calls over shared primitives                            | `Brain/docs/brain-architecture.md`         |
| D29 | Multi user from the start, fully isolated                                        | `High-level/docs/platform-architecture.md` |
| D30 | Notification delivery is its own service                                         | `High-level/docs/platform-architecture.md` |
| D31 | Authentication is built by hand, and registration is invite coded                | `Core/docs/core-architecture.md`           |

## Questions

| #   | Title                                                | Lives in                            |
| --- | ---------------------------------------------------- | ----------------------------------- |
| Q1  | What goes in a memory event                          | `Brain/docs/open-questions.md`      |
| Q2  | The trust model                                      | `Brain/docs/open-questions.md`      |
| Q3  | The retrieval interface                              | `Brain/docs/open-questions.md`      |
| Q4  | Review queue lifecycle                               | `Brain/docs/open-questions.md`      |
| Q5  | Does document memory survive                         | `Brain/docs/open-questions.md`      |
| Q6  | What consolidation actually does in v1 of the brain  | `Brain/docs/open-questions.md`      |
| Q7  | Core in Go or C#                                     | `Core/docs/open-questions.md`       |
| Q8  | What the feature manifest contains                   | `Core/docs/open-questions.md`       |
| Q9  | Routing implementation                               | `Core/docs/open-questions.md`       |
| Q10 | Does projects live in the core codebase              | `Core/docs/open-questions.md`       |
| Q11 | Temporal                                             | `High-level/docs/open-questions.md` |
| Q12 | Shell UX ambition                                    | `High-level/docs/open-questions.md` |
| Q13 | The feature list                                     | `High-level/docs/open-questions.md` |
| Q14 | Mobile                                               | `High-level/docs/open-questions.md` |
| Q15 | Scope and confirmation on immediate writes           | `Brain/docs/open-questions.md`      |
| Q16 | Do decisions get a first class side table            | `Brain/docs/open-questions.md`      |
| Q17 | How the knowledge model represents level             | `Brain/docs/open-questions.md`      |
| Q18 | How user isolation is enforced                       | `High-level/docs/open-questions.md` |
| Q19 | What language the notification service is written in | `High-level/docs/open-questions.md` |
