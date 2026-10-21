---
name: code-review
description: A structured process for reviewing code changes the way a careful senior engineer would, and for responding to review feedback. Covers understanding the change, what to check and in what order (correctness, security, design, tests, readability), judging severity, writing clear and respectful comments, deciding whether to approve, reviewing special kinds of changes, and producing a review report. Use this skill whenever asked to review, check, audit, or give feedback on code, a diff, a pull request, a merge request, a commit, or a branch, whenever self-reviewing work before declaring it finished, and whenever responding to review comments, even if the request is phrased casually, such as "look over this" or "is this okay?".
---

# Code Review

Code review has two purposes: catching problems before they reach users, and keeping the codebase healthy over time. A good review finds what matters, explains it clearly, and helps the change land. It does not nitpick, rewrite the author's work to personal taste, or approve something just because it is large and tiring to read.

The standard for approval, from Google's engineering practices: **approve a change once it definitely improves the overall health of the codebase, even if it isn't perfect.** No change is perfect; the question is whether this one makes things better and introduces no serious problems.

Read the project's `AGENTS.md` first. Review against the project's conventions, tools, and architecture, not personal preference. Where the project's conventions differ from genski's defaults, the project's conventions are correct for that project.

## Before you start

1. **What is this change meant to do?** Read the description, linked issue, and commit messages. If the intent is unclear, that is the first piece of feedback.
2. **What does it touch?** Get an overview of the files changed before reading lines. Identify the most important or risky parts.
3. **What context do I need?** Read surrounding code, callers, and related tests, not only the changed lines. Many bugs are in what the diff *doesn't* show.
4. **Can I run it?** If tools are available, run the tests, linter, and type checks. Never report that tests pass without running them.

## The review, in order

Review in order of importance. Problems higher on this list matter more, and finding one may make lower-level comments pointless (there's no value in polishing names in code that must be redesigned).

### 1. Intent and approach

- Does the change do what it claims, and is that the right thing to do?
- Is the overall approach sound, or is there a significantly simpler or safer way?
- Is the change the right size? Very large changes should usually be split. Unrelated changes mixed in should be separated.

### 2. Correctness

- Does the logic work for normal cases, edge cases, empty and maximum inputs, and invalid input?
- Are errors handled properly (see `error-handling`)? Is anything swallowed, leaked, or left half-done?
- Concurrency: could two requests, jobs, or users interfere? Are operations that must be atomic actually atomic?
- Data: will this work with existing data? Are migrations safe? Could anything be lost or corrupted?
- Does it break any existing callers, APIs, or contracts?

### 3. Security

Apply the `security` skill's checklist to anything touching input, identity, data, or network exposure. Pay particular attention to:

- Missing permission or ownership checks.
- Input that reaches a query, command, file path, or HTML without protection.
- Secrets or sensitive data in code, logs, or responses.
- New dependencies, and anything loosened or disabled.

### 4. Design

Apply the `software-design` skill:

- Is code in the right module and layer? Does it respect module boundaries and data ownership?
- Is it as simple as the problem allows? Watch for unnecessary abstractions, speculative flexibility, and premature generalisation.
- Does it fit the existing architecture and patterns, or introduce a competing one?
- Does it contradict a recorded decision?

### 5. Tests

Apply the `testing` skill:

- Are there tests for the new or changed behaviour, including failure paths?
- Do the tests check behaviour, or are they tied to implementation details?
- Would the tests fail if the code were broken? A test that can't fail is worthless.
- For bug fixes, is there a test that reproduces the bug?
- Were any tests deleted, skipped, or weakened? If so, why?

### 6. Readability and maintainability

Apply the `code-quality` skill:

- Can someone unfamiliar with the change understand it? Are names precise?
- Is anything confusing that deserves a comment explaining *why*?
- Is there leftover debug code, dead code, or commented-out code?
- Is it consistent with the codebase's conventions?

### 7. Everything else

Documentation updated where behaviour changed, performance concerns in hot paths, operational concerns (logging, configuration, monitoring), and accessibility for interface changes.

Leave formatting and style to automated tools. If the project has a formatter and linter, don't comment on what they enforce.

For a detailed checklist for each area, read `references/checklist.md`.

## Judging severity

Label every comment, so the author knows what must change and what is optional.

| Label | Meaning | Examples |
|---|---|---|
| **Blocking** | Must be fixed before merging | Bugs, security issues, data loss risk, broken tests, missing tests for critical behaviour, architecture violations |
| **Should fix** | Important, but could be handled in a follow-up with agreement | Missing edge-case tests, unclear naming of central concepts, avoidable complexity |
| **Suggestion** | Optional improvement; the author decides | Alternative approaches, minor readability improvements |
| **Nit** | Trivial; take it or leave it | Small wording or naming preferences not covered by tooling |
| **Question** | Seeking understanding, not requesting change | "Why is this retried here but not in the other handler?" |
| **Praise** | Something done well | A clear test, a well-chosen simplification |

Be honest about severity. Marking preferences as blocking erodes trust; marking real bugs as suggestions lets them through.

## Writing comments

- **Be specific:** point to the exact line and the exact problem.
- **Explain why:** state the consequence, not just the rule. "This query isn't scoped to the user, so any signed-in user can read any invoice by changing the ID" is better than "Add an auth check".
- **Suggest a fix** where you can, ideally with a short code example.
- **Critique the code, never the person.** "This function handles three concerns" rather than "You made this too complicated".
- **Ask when unsure.** If you might be missing context, ask a question rather than asserting a problem.
- **Don't repeat yourself.** If the same issue appears in many places, comment once and note that it applies elsewhere.
- **Acknowledge good work.** Brief, genuine praise for good decisions helps people know what to keep doing.

For examples of strong and weak comments, read `references/comments.md`.

## Deciding the outcome

- **Approve** when the change improves the codebase and has no blocking issues. Non-blocking comments can be addressed or not, at the author's discretion.
- **Request changes** when there is at least one blocking issue. Say clearly which comments are blocking.
- **Comment without deciding** when you have questions whose answers determine whether there's a problem.

Don't withhold approval for personal preferences, for issues that existed before the change, or for improvements unrelated to the change. Suggest those as follow-ups instead.

## The review report

When reporting a review in conversation or as a summary, use this structure:

```
## Summary
One or two sentences: what the change does and your overall assessment.

## Verdict
Approve / Request changes / Questions before deciding

## Blocking
- [file:line] Problem, why it matters, suggested fix.

## Should fix
- [file:line] ...

## Suggestions and nits
- [file:line] ...

## Questions
- ...

## What's done well
- ...

## Verification
What you ran (tests, linter, type checks) and the results, or what you could not run.
```

Leave out empty sections. If there are no blocking issues, say so explicitly.

## Self-review

Before declaring your own work finished, review your own change using this skill, as if someone else wrote it. Read the full diff, not just the parts you remember changing. Self-review catches leftover debug code, missing tests, and unintended changes before anyone else has to.

## Responding to review feedback

When your work receives review comments:

- **Take every comment seriously.** Consider whether it's right before responding. Reviewers often see what the author can't.
- **Fix what's correct,** and say briefly what you changed.
- **Disagree with reasons** when you believe a comment is wrong, and explain the trade-off. Don't silently ignore comments, and don't make a change you believe is harmful just to close the discussion.
- **Ask for clarification** when a comment is ambiguous.
- **Don't widen the scope.** Suggestions for unrelated improvements can become follow-up work.
- **Re-review your own fixes** before asking for another review, so new issues aren't introduced.

## Common mistakes

- Reviewing only the changed lines, without the surrounding context.
- Commenting on style and formatting while missing a logic bug or security issue.
- Approving large changes without reading them properly.
- Marking personal preferences as blocking, or real bugs as optional.
- Vague comments ("this is confusing", "fix this") with no reason or suggestion.
- Requesting rewrites to personal taste when the existing approach is sound.
- Blocking a change because of problems that existed before it.
- Reporting that tests pass without running them.
- Missing deleted or weakened tests.
- Only criticising, never noting what was done well.

## Reference files

| File | Read when |
|---|---|
| `references/checklist.md` | Doing a thorough review and needing detailed questions for each area |
| `references/comments.md` | Writing review comments, or responding to them |
| `references/special-cases.md` | Reviewing large changes, refactors, migrations, dependency updates, configuration, generated code, or AI-written code |

## Sources

Google Engineering Practices, *Code Review Developer Guide*; Karl Wiegers, *Peer Reviews in Software*; Microsoft Research, "Expectations, Outcomes, and Challenges of Modern Code Review" (Bacchelli and Bird); SmartBear, "Best Kept Secrets of Peer Code Review"; Conventional Comments (conventionalcomments.org); OWASP Code Review Guide.