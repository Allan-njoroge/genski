---
name: decisions
description: A method for making, escalating, and recording significant technical decisions, in any project. Covers recognising which decisions are significant, identifying the quality attributes that matter, comparing options and their trade-offs, judging reversibility, deciding when to decide, knowing which decisions to bring to a person rather than make alone, taking on technical debt deliberately, and writing Architecture Decision Records (ADRs). Use this skill whenever choosing a library, framework, database, service, pattern, data model, API shape, or architecture; whenever a task has several reasonable approaches with different trade-offs; when about to make a change that is hard to undo; when asked "should we use X or Y?" or "how should we build this?"; and when writing or reading decision records.
---

# Decisions

Architecture is largely the set of decisions that are expensive to change later. Most technical decisions are small and can be made quickly. A few shape the system for years: which database, how modules are divided, the shape of a public API, how data is modelled. Good engineering is knowing which decisions are which, giving the important ones proper thought, and leaving a record so future readers, human or agent, understand why things are the way they are.

Read the project's `AGENTS.md` first, and check for existing decision records (often in `docs/adr/`, `docs/decisions/`, or similar). A recorded decision is the project's chosen direction. Follow it, and if you believe it should change, raise that rather than quietly working around it.

## Before you start

1. **Is this decision significant?** Use the tests in rule 1. Most aren't; make those quickly and move on.
2. **Has it already been decided?** Check decision records, `AGENTS.md`, and existing patterns in the code.
3. **Is it mine to make?** Use rule 7 to tell whether to decide, or to present options to the person you're working with.

## Core rules

### 1. Recognise significant decisions

A decision is significant if one or more is true:

- **It's hard to reverse:** changing it later means migrating data, rewriting many files, or breaking clients.
- **It's wide-reaching:** it affects many modules, teams, or users.
- **It sets a precedent:** others will copy it, such as the first use of a pattern or library.
- **It's costly:** in money, performance, operational effort, or security exposure.
- **It's contested:** reasonable people would choose differently.

Examples: choosing a database or message broker, adding a framework or major library, dividing modules or services, designing a public API, choosing an authentication approach, defining a core data model, and choosing between building something or using a service.

Not significant: naming a private function, choosing between two equivalent standard-library calls, or the internal structure of one small module. Decide these quickly, following the codebase's conventions.

### 2. Name what matters most

Every system involves trade-offs. You can't maximise everything, so first identify which qualities matter most for *this* system and *this* decision:

- **Performance:** speed, throughput, efficiency.
- **Scalability:** handling growth in users, data, or load.
- **Reliability:** staying available and correct when things fail.
- **Security:** protecting data and access.
- **Maintainability:** ease of understanding and changing the code.
- **Simplicity:** fewer moving parts, less to learn.
- **Cost:** money, infrastructure, and people's time.
- **Speed of delivery:** how quickly it can be built and shipped.
- **Operability:** ease of deploying, monitoring, and debugging.
- **Compatibility:** fitting with existing systems, skills, and clients.

Rank the two or three that matter most here. That ranking decides most trade-offs.

For questions to identify these and common tensions between them, read `references/quality-attributes.md`.

### 3. Compare real options

For a significant decision, consider at least two genuine options, and include "do nothing" or "the simplest thing" where reasonable. For each option, state:

- How well it serves the qualities that matter most.
- Its costs and risks, including operational costs and learning required.
- What it rules out or makes harder later.
- How reversible it is.

Avoid deciding first and justifying afterwards. If one option is clearly best, the comparison is quick, but do it.

For a method of comparing options, read `references/evaluating-options.md`.

### 4. Match care to reversibility

- **Reversible decisions** (two-way doors) can be made quickly, with less information, and corrected later. Most decisions are like this.
- **Irreversible or costly-to-reverse decisions** (one-way doors) deserve careful thought, more evidence, and agreement from the people affected.

Where possible, **make decisions more reversible**: put a volatile choice behind an interface you own, keep data in portable formats, introduce new approaches gradually with a feature flag, or try a small proof of concept before committing.

### 5. Decide at the right time

Don't make decisions earlier than needed. Deciding before you understand the problem locks in guesses. But don't delay past the point where delay causes harm. Aim for the **last responsible moment**: when you know enough to decide well, before the lack of a decision blocks progress or makes things expensive.

When the right answer depends on unknowns, reduce the uncertainty: build a small, throwaway experiment, measure, or ask.

### 6. Prefer boring, proven choices

New and fashionable technology has hidden costs: unknown failure modes, immature tools, few people who understand it, and more to operate. Choose established, well-understood technology unless a new option offers a clear, significant advantage for a quality that matters most. Prefer what the project and team already use and know.

### 7. Know which decisions to bring to a person

As an agent, make routine decisions yourself, following the project's conventions. Bring these to the person you are working with, with options and a recommendation, instead of deciding alone:

- Adding a new framework, major library, database, or external service.
- Changing a public API, data model, or database schema in a way that affects existing clients or data.
- Introducing a new architectural pattern or changing module boundaries.
- Anything with significant cost, security, legal, or privacy implications.
- Anything irreversible, such as deleting data or making a breaking change.
- Going against an existing decision record or established pattern.
- Choices where the right answer depends on business priorities you don't know.

When presenting the decision, be brief: the options, the key trade-off between them, your recommendation, and why. Then let them choose.

### 8. Take on technical debt deliberately

Sometimes a quick, imperfect solution is the right call, for example to meet a deadline or test an idea. That's fine if it's a conscious choice:

- Say clearly that it's a shortcut, what the risk is, and what a proper solution would look like.
- Record it where the project tracks such things (a ticket, a decision record, or a `TODO` with a reference).
- Contain it, so it doesn't spread into other code.

Accidental shortcuts that nobody knows about are the dangerous kind.

### 9. Record significant decisions

Write an Architecture Decision Record (ADR) for each significant decision: a short document capturing the context, the decision, the options considered, and the consequences. ADRs let future readers understand why, so they don't undo a good decision out of ignorance or keep a bad one out of caution.

- Keep each ADR short, usually under a page.
- Number them, store them with the code, and never rewrite history: when a decision changes, write a new ADR that supersedes the old one.
- Write them when the decision is made, not months later.

For the template, examples, and guidance, read `references/adr.md`.

### 10. Revisit decisions when the context changes

Decisions are made for a context: the team, the scale, the requirements, the available technology. When the context changes significantly, an old decision may no longer be right. Record new evidence, propose a change, and write a new ADR if the decision is revised.

## Example

A task asks for background email sending in a project that has no job queue.

**Poor:** the agent installs a new message broker, adds a job framework, writes a worker service, and changes deployment, all without asking, because it's the "proper" way to do background jobs.

**Better:** the agent recognises this as significant (a new piece of infrastructure, operational cost, a precedent for all future background work) and presents it:

```
Sending email in the background needs a decision, since the project has no job queue yet.

Options:
1. Database-backed job table, processed by a worker in the same application.
   Simple, no new infrastructure; fine at current volume. Less suitable for very high throughput.
2. A dedicated queue (e.g. Redis-based or a managed queue service).
   Scales well and is widely used; adds infrastructure to run, monitor, and pay for.
3. Send directly during the request, with a retry.
   No new parts; but slow and fragile if the email provider is down.

Recommendation: option 1. It meets current needs with the least new infrastructure, and if
we put jobs behind a small interface, moving to a queue later is straightforward.

Shall I go ahead with option 1? If so, I'll record it in an ADR.
```

## Common mistakes

- Treating small decisions as big ones, and spending a long time on choices that are easy to change.
- Treating big decisions as small ones, adding infrastructure or frameworks without discussion.
- Considering only one option, or deciding first and justifying later.
- Choosing new, fashionable technology without a clear advantage.
- Designing for scale or flexibility the system doesn't need.
- Working around a recorded decision instead of raising it.
- Leaving no record, so the reasoning is lost.
- Rewriting old ADRs instead of superseding them.
- Taking shortcuts silently, without recording the debt.

## Reference files

| File | Read when |
|---|---|
| `references/quality-attributes.md` | Identifying which qualities matter for a system, or weighing trade-offs between them |
| `references/evaluating-options.md` | Comparing options for a significant decision, reducing uncertainty, or handling technical debt |
| `references/adr.md` | Writing, reading, or superseding an Architecture Decision Record |

## Sources

Michael Nygard, "Documenting Architecture Decisions" (2011); ISO/IEC 25010, *Systems and software quality models*; Len Bass, Paul Clements, and Rick Kazman, *Software Architecture in Practice*; Mark Richards and Neal Ford, *Fundamentals of Software Architecture*; Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani, *Software Architecture: The Hard Parts*; Mary and Tom Poppendieck, *Lean Software Development* (the last responsible moment); Jeff Bezos, 2015 shareholder letter (one-way and two-way doors); Dan McKinley, "Choose Boring Technology"; Martin Fowler, "Technical Debt Quadrant".