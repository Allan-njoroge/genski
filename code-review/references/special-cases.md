# Reviewing Special Kinds of Changes

Some changes need a different focus from ordinary feature work. This file covers what to look for in each.

## Contents

1. Large changes
2. Refactoring
3. Database migrations
4. Dependency updates
5. Configuration and infrastructure
6. Generated code
7. AI-written code
8. Security-sensitive changes
9. Bug fixes
10. Deleted code

## 1. Large changes

Review quality drops sharply as change size grows; beyond a few hundred lines, reviewers start skimming.

- If a change is too large to review properly, say so, and suggest how to split it (for example, by layer, by feature slice, or refactor first then feature).
- If splitting isn't possible, review in a deliberate order: the design and public interfaces first, then the riskiest logic, then tests, then the rest.
- Ask the author for a walkthrough or a summary of where to focus.
- Never approve a large change you haven't actually reviewed.

## 2. Refactoring

A refactor should change structure without changing behaviour.

- Confirm that behaviour hasn't changed. Look for subtle differences: changed defaults, error handling, ordering, or edge cases.
- Check that tests existed before the refactor and still pass without modification. Tests changed alongside a refactor need scrutiny: are they being adapted to new structure, or to new behaviour?
- Refactoring and behaviour changes should be in separate commits or pull requests. If they're mixed, ask for them to be separated.

## 3. Database migrations

Migrations can lose data and are hard to reverse in production.

- Does the migration work on the real volume of data? Operations that rewrite or lock large tables can cause downtime.
- Is it compatible with the currently deployed code, which keeps running while the migration applies?
- Are destructive steps (dropping columns or tables, narrowing types) separated from the code that stops using them, so the old code isn't broken mid-deployment?
- Is data being transformed correctly, including unusual existing rows (nulls, duplicates, legacy formats)?
- Is there a way back, either a reverse migration or a documented recovery plan?
- Are new indexes created in a way that won't lock the table, where the database supports it?

## 4. Dependency updates

- Is the new dependency or version necessary?
- For new packages: is it legitimate, maintained, widely used, and correctly named (not an imitation)? Is the version mature rather than just released?
- For major version updates: check the changelog for breaking changes, and confirm the code was adapted.
- Is the lock file updated and committed? Are there unexpected changes to other packages in it?
- Does the vulnerability audit pass?

## 5. Configuration and infrastructure

- Are production values safe: debug off, minimal permissions, restrictive network access?
- Are secrets referenced from a secret store rather than written in?
- Do changes apply to the intended environments only?
- Could the change cause downtime, and is there a rollback plan?
- Are permissions as narrow as possible? Watch for wildcards in access policies.

## 6. Generated code

Code produced by tools (API clients, schema types, protocol buffers):

- Review the source (the schema or specification) and the generator configuration, not the generated output line by line.
- Check that generated files are produced by the documented command and haven't been hand-edited.
- Confirm the project's convention on whether generated code is committed.

## 7. AI-written code

Code written by AI agents, including your own output when self-reviewing, has characteristic risks. Review it with the same standards as human code, and pay extra attention to:

- **Invented APIs:** functions, options, or packages that don't exist or don't behave as assumed. Verify unfamiliar calls against the real documentation or source.
- **Non-existent or imitation packages:** confirm every new dependency is the real, intended one.
- **Plausible but wrong logic:** code that looks right and reads well but mishandles edge cases. Check edge cases deliberately.
- **Over-engineering:** extra layers, abstractions, configuration, and defensive code nobody needs.
- **Scope creep:** unrelated files reformatted, renamed, or "improved".
- **Weakened tests:** assertions changed to match the code, tests skipped or deleted, or tests that can't fail.
- **Disabled safeguards:** security checks, type checks, or linter rules turned off to make things work.
- **Inconsistent patterns:** a new way of doing something the codebase already does differently.
- **Unverified claims:** descriptions saying tests pass, or that something was checked, without evidence.

## 8. Security-sensitive changes

Changes to authentication, authorisation, cryptography, payment, personal data, or anything internet-facing deserve the most careful review.

- Apply the full security section of `checklist.md`.
- Think like an attacker: how would you misuse this feature?
- Look for what's missing, such as an absent permission check, not just what's wrong in what's present.
- Recommend a second reviewer with security experience for high-risk changes, where the team has one.

## 9. Bug fixes

- Does the fix address the root cause, or only a symptom?
- Is there a regression test that fails without the fix?
- Could the same bug exist elsewhere in similar code?
- Does the fix change behaviour that other code relies on?

## 10. Deleted code

Deletions are usually welcome, but check:

- Is the code truly unused? Search for callers, including dynamic references, configuration, scheduled jobs, and other repositories that may depend on it.
- Were tests removed along with it, and only those that tested the deleted behaviour?
- Does any documentation or configuration still refer to it?