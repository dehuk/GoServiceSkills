---
name: architecture-decision-records
description: Capture architectural decisions as numbered ADR files in docs/adr/NNNN-title.md with context, alternatives, and consequences, and look up existing ADRs to explain why the system is shaped the way it is. Use whenever a choice between significant alternatives is made or discussed (database, broker, sync vs async integration, service boundaries, API style, auth strategy), when the user says "record this decision" or "ADR this", or asks "why did we choose X?" — even if the user does not mention ADRs.
metadata:
  origin: ECC
---

# Architecture Decision Records

An ADR is a short document that records one architectural decision, the forces behind it, and what was rejected. Decisions that live only in chat threads and PR comments get relitigated, because nobody can find out why the code is shaped this way. ADRs stored next to the code fix that.

## When to Write an ADR

Write one when a decision is costly to reverse or affects more than one module or service:

| Category | Examples |
|----------|----------|
| Technology choices | Database, message broker, cache, major library |
| Service boundaries | Splitting or merging services, which service owns which data |
| Integration style | Sync HTTP vs async events, choreography vs orchestration |
| Data | Schema strategy, consistency model, caching, retention |
| API | Versioning scheme, error format, pagination style, auth mechanism |
| Reliability | Outbox, idempotency strategy, retry and timeout policy |
| Infrastructure and process | Deployment model, observability stack, release strategy |

Skip trivial or easily reversible choices (naming, formatting, a helper's signature). Recording them buries the decisions that matter.

### Detection signals

- Explicit: "let's go with X", "we should use X instead of Y", "the trade-off is worth it because...", "ADR this".
- Implicit: two options compared and one chosen with a rationale; a planning discussion that settles a trade-off. Suggest an ADR here, but do not create one without the user's confirmation.
- Questions: "why did we choose X?", "why is this async?" — read existing ADRs before answering.

## Template

```markdown
# ADR-NNNN: [Decision Title]

**Date**: YYYY-MM-DD
**Status**: proposed | accepted | deprecated | superseded by ADR-NNNN
**Deciders**: [who was involved]

## Context

[2-5 sentences: the problem, constraints, and forces at play]

## Decision

[1-3 sentences, present tense: "We use X for Y."]

## Alternatives Considered

### [Alternative name]
- **Pros**: ...
- **Cons**: ...
- **Why not**: [the specific reason it lost]

## Consequences

### Positive
- ...

### Negative
- ...

### Risks
- [risk and mitigation]
```

## Layout

Store ADRs at `docs/adr/NNNN-title.md`: four-digit, zero-padded, never reused numbers, kebab-case title. Zero padding keeps them sorted in file listings; stable numbers keep links valid.

```
docs/adr/
├── README.md                          index of all ADRs
├── template.md                        blank template for manual use
├── 0001-postgres-as-primary-store.md
├── 0002-nats-jetstream-for-events.md
└── 0003-transactional-outbox.md
```

Index format (`docs/adr/README.md`):

```markdown
# Architecture Decision Records

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [0001](0001-postgres-as-primary-store.md) | PostgreSQL as primary datastore | accepted | 2026-01-15 |
| [0002](0002-nats-jetstream-for-events.md) | NATS JetStream for domain events | accepted | 2026-01-20 |
```

## Workflow

### Recording a decision

1. If `docs/adr/` does not exist, ask before creating it with `README.md` (index header) and `template.md`. Never create files without consent.
2. Extract the decision, the context that forced it, and the alternatives with the reason each lost.
3. Number it: scan `docs/adr/` and take the highest number plus one.
4. Show the draft to the user. Write `docs/adr/NNNN-title.md` only after approval; if declined, discard the draft.
5. Append a row to the index.
6. If it supersedes an earlier ADR, set the old one's status to `superseded by ADR-NNNN`. Do not delete or rewrite the old ADR; its history is the point.

### Answering "why did we choose X?"

1. If `docs/adr/` is missing, say so and offer to start recording decisions.
2. Otherwise scan the index, read matching ADRs, and summarize Context and Decision, noting the status (a superseded ADR is not current policy).
3. If nothing matches, say so and offer to record the decision now.

## What Makes a Good ADR

- **Specific**: "Use pgx v5 with pgxpool", not "use a Postgres driver".
- **The why over the what**: the rationale is what future readers lack.
- **Real alternatives**: "we just picked it" is not a rationale. Include the option that almost won.
- **Honest consequences**: every decision has a cost; name it.
- **Short**: readable in two minutes. If Context runs past ~10 lines, it is an essay, not an ADR.
- **Backfills marked**: when recording a past decision, state the original date and that it was written after the fact.

## Lifecycle

```
proposed -> accepted -> [deprecated | superseded by ADR-NNNN]
```

- **proposed**: under discussion, not yet in effect.
- **accepted**: in effect; code should follow it.
- **deprecated**: no longer relevant (for example, the feature was removed).
- **superseded**: replaced; always link the replacement.

## Related

- The **architect** and **planner** agents should propose an ADR when their designs settle a significant trade-off.
- Reviewers (for example the **go-reviewer** agent) can flag changes that introduce an architectural shift without an ADR.
