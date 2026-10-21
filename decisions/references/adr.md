# Architecture Decision Records

An Architecture Decision Record (ADR) is a short document recording one significant decision: the context, what was decided, and what follows from it. The format was popularised by Michael Nygard in 2011.

## Contents

1. Why ADRs matter
2. When to write one
3. Where to keep them
4. Template
5. Writing guidelines
6. Lifecycle and statuses
7. Example
8. Reading ADRs as an agent

## 1. Why ADRs matter

Without a record, the reasons behind decisions are lost when people forget or leave. Newcomers, human or AI, then either:

- undo a good decision because they don't understand it, or
- keep a bad decision because they're afraid to change something they don't understand.

ADRs make the reasoning available, so decisions can be respected or revisited on their merits.

## 2. When to write one

Write an ADR for decisions that are significant (see `SKILL.md`, rule 1), for example:

- Choosing a framework, database, message broker, or major library.
- Choosing an architecture style or changing module boundaries.
- Designing a public API, an authentication approach, or a core data model.
- Adopting a convention that everyone must follow.
- Deliberately taking on significant technical debt.
- Reversing or changing an earlier decision.

Don't write ADRs for routine choices that follow existing conventions.

## 3. Where to keep them

- Store ADRs in the repository, with the code, so they're versioned and reviewed like code. A common location is `docs/adr/` or `docs/decisions/`. Follow the project's convention.
- Name them with a sequence number and a short title: `0007-use-database-backed-job-queue.md`.
- Keep an index (a README in the folder, or a list) if there are many.
- Mention the ADR folder in `AGENTS.md` so agents know to check it.

## 4. Template

```markdown
# [Number]. [Short title of the decision]

- **Status:** Proposed | Accepted | Deprecated | Superseded by [ADR number]
- **Date:** YYYY-MM-DD
- **Deciders:** [people or roles involved]

## Context

What situation or problem requires a decision? What constraints apply, and which
qualities matter most? State the facts, without arguing for an option yet.

## Options considered

1. **[Option A]:** short description. Benefits; drawbacks.
2. **[Option B]:** short description. Benefits; drawbacks.
3. **[Option C]:** ...

## Decision

What we decided, stated clearly: "We will ..."

## Consequences

What becomes easier, what becomes harder, and what new risks or obligations follow.
Include follow-up actions, and the conditions under which this decision should be revisited.
```

## 5. Writing guidelines

- **Keep it short,** usually under one page. An ADR that takes an hour to read won't be read.
- **One decision per ADR.**
- **State the context neutrally,** then the decision clearly ("We will…"), so there's no doubt what was chosen.
- **Be honest about consequences,** including the negative ones. Every real decision has some.
- **Write it when the decision is made,** while the reasoning is fresh.
- **Use plain language,** so someone new to the project can follow it.

## 6. Lifecycle and statuses

| Status | Meaning |
|---|---|
| **Proposed** | Under discussion, not yet agreed |
| **Accepted** | Agreed and in effect |
| **Deprecated** | No longer recommended, but not replaced |
| **Superseded** | Replaced by a later ADR, which is referenced |

Rules:

- **Don't rewrite accepted ADRs.** If a decision changes, write a new ADR that explains the new context and decision, and mark the old one "Superseded by [new number]". The history of reasoning is part of the value.
- Small corrections (typos, broken links) are fine.

## 7. Example

```markdown
# 0007. Use a database-backed job queue for background work

- **Status:** Accepted
- **Date:** 2026-10-21
- **Deciders:** Backend team

## Context

We need to send emails and generate reports without making users wait. We currently
have no background job system. Volume is low (a few thousand jobs per day) and expected
to grow moderately. The team is small and wants to avoid running additional infrastructure.
Reliability matters most: jobs must not be lost if the application restarts.

## Options considered

1. **Jobs table in our existing database, processed by a worker in the application.**
   No new infrastructure; jobs survive restarts; transactional with our data. Limited
   throughput compared with a dedicated queue.
2. **Managed message queue service.** Scales well, widely used. Adds a service to
   configure, monitor, and pay for; jobs aren't transactional with our database.
3. **Send work directly during the request.** Nothing new to build. Slow responses, and
   work is lost if the provider fails.

## Decision

We will use a jobs table in the existing database, processed by a worker running within
the application. Jobs are accessed through a small `JobQueue` interface.

## Consequences

- No new infrastructure to run or pay for.
- Jobs can be created in the same transaction as the data they relate to.
- Throughput is limited; if we exceed roughly 100,000 jobs per day, we should revisit
  this decision. The `JobQueue` interface makes moving to a dedicated queue straightforward.
- We need to add monitoring for stuck and failed jobs.
```

## 8. Reading ADRs as an agent

- Before making structural changes, check the project's ADR folder for relevant decisions.
- Follow accepted decisions, even if you'd have chosen differently.
- If a task seems to require going against an accepted ADR, stop and raise it with the person you're working with, explaining why, rather than working around it.
- If you notice an ADR's revisit conditions have been met (for example, the volume limit has been reached), mention it.
- When you help make a significant decision, offer to write the ADR for it.