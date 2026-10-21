# Evaluating Options

This file gives a method for comparing options for a significant decision, reducing uncertainty, and handling shortcuts responsibly.

## Contents

1. A lightweight method
2. Comparing options
3. Reversibility
4. Reducing uncertainty
5. Build, buy, or use what exists
6. Technical debt
7. Presenting a decision to a person

## 1. A lightweight method

For a significant decision:

1. **State the problem** in one or two sentences, including constraints.
2. **Identify the two or three qualities that matter most** (see `quality-attributes.md`).
3. **List the realistic options,** including the simplest one, and "change nothing" if that's viable.
4. **Compare them** against those qualities, plus cost, risk, and reversibility.
5. **Choose,** or recommend and ask, depending on who should decide (see `SKILL.md`, rule 7).
6. **Record it** in an ADR (see `adr.md`).

For smaller but still non-trivial decisions, steps 1 to 4 can happen in a few minutes in your head. The value is in not skipping them.

## 2. Comparing options

A simple comparison table is usually enough:

| | Option A: job table | Option B: managed queue | Option C: send in request |
|---|---|---|---|
| Simplicity | ✅ No new infrastructure | ⚠️ New service to learn and run | ✅ Nothing new |
| Reliability | ✅ Retries, survives restarts | ✅ Strong | ❌ Fails if provider is down |
| Scalability | ⚠️ Fine for current volume | ✅ High | ❌ Slows requests |
| Cost | ✅ None | ⚠️ Monthly cost | ✅ None |
| Reversibility | ✅ Easy to move to a queue later | ⚠️ Harder to remove | ✅ Easy |

Guidelines:

- Compare on the qualities that matter most for this decision, not every possible one.
- Be honest about each option's weaknesses, including the one you prefer.
- Include costs that are easy to forget: operations, monitoring, learning, licences, and vendor lock-in.
- Avoid scoring systems with false precision. The table is for seeing trade-offs, not calculating a winner.

## 3. Reversibility

Ask: if this turns out wrong, what would changing it cost?

| Easy to reverse | Hard to reverse |
|---|---|
| Internal implementation details | Public API shape |
| A library hidden behind your own interface | A library used directly throughout the code |
| Feature-flagged behaviour | Data model and stored data formats |
| Configuration | Choice of primary database |
| A new module | Splitting into separate services |
| | Deleting data |

Ways to make decisions more reversible:

- **Wrap volatile choices** behind an interface you own, so they can be swapped.
- **Introduce gradually,** behind a feature flag or for a subset of traffic.
- **Keep data portable,** in standard formats, with a clear schema.
- **Run old and new in parallel** during a migration, so you can switch back.
- **Prototype first,** and throw the prototype away.

## 4. Reducing uncertainty

When the right choice depends on something unknown, find out cheaply before committing:

- **Spike:** a short, time-boxed experiment to answer one question ("Can this library handle our file sizes?"). Throw the code away afterwards.
- **Proof of concept:** a minimal end-to-end version of the riskiest part.
- **Measurement:** benchmark the real workload instead of guessing about performance.
- **Ask:** the person you're working with, the team, or the people who will use or run it often know what you don't.
- **Read:** official documentation, known limitations, and experience reports from others using it at a similar scale.

## 5. Build, buy, or use what exists

Before building something, check:

1. **Does the project already have it?** Reuse existing code, libraries, and services.
2. **Does the standard library or platform provide it?**
3. **Is there a well-established library or service** for it?
4. **Is it core to what makes the product valuable?** Build what differentiates the product; use existing solutions for everything else (authentication, payments, email delivery, and similar).

Prefer building when existing options don't fit the important requirements, the dependency risk is high, or it's genuinely simple and stable. Prefer existing solutions when the problem is complex, security-sensitive, or well solved by others.

## 6. Technical debt

Technical debt is the future cost of choosing a quicker or easier solution now. Martin Fowler's technical debt quadrant distinguishes debt that is deliberate or inadvertent, and prudent or reckless:

| | Reckless | Prudent |
|---|---|---|
| **Deliberate** | "We don't have time for design." | "We must ship now and will deal with the consequences." |
| **Inadvertent** | "What's layering?" | "Now we know how we should have done it." |

Aim only for **prudent, deliberate** debt, and learn from the inadvertent kind.

When taking on debt deliberately:

- Make it visible: state that it's a shortcut, what the risk is, and what the proper solution is.
- Record it in the project's tracking system, an ADR, or a referenced `TODO`.
- Contain it, so other code doesn't build on the shortcut.
- Agree when it will be repaid, if it carries real risk.

## 7. Presenting a decision to a person

When a decision should be made by the person you're working with, keep it short and decision-ready:

```
[One sentence: what needs deciding and why it matters.]

Options:
1. [Option]: [main benefit]; [main cost or risk].
2. [Option]: [main benefit]; [main cost or risk].
3. [Option, if relevant].

Recommendation: [option], because [reason tied to what matters most].

[Question: shall I proceed with the recommendation?]
```

Don't bury the recommendation, don't list every possible option, and don't proceed with an irreversible choice before they answer.