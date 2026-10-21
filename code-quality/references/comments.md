# Comments and Documentation

Comments should add information that code cannot express. The best code needs few comments to explain *what* it does, but most non-trivial code benefits from comments explaining *why*.

## Contents

1. Comments worth writing
2. Comments to avoid
3. Documenting public interfaces
4. TODOs
5. Keeping comments true

## 1. Comments worth writing

- **Intent and reasons:** why the code takes this approach, especially when the obvious approach was rejected.
  `# Process in batches of 500: the provider rejects larger requests.`
- **Business rules:** the rule behind the code, ideally with its source.
  `# Refunds are allowed for 30 days after delivery (Customer Terms, section 4).`
- **Warnings:** consequences that aren't visible.
  `# Not thread-safe: callers must hold the account lock.`
- **Workarounds:** what is being worked around, and when it can be removed.
  `# Workaround for library bug #1234; remove after upgrading to v3.`
- **Non-obvious behaviour:** units, formats, edge cases, assumptions.
  `# Timestamps are in UTC; amounts are in the smallest currency unit (cents).`
- **Summaries of complex algorithms**, with a reference to the source if one exists.

## 2. Comments to avoid

- **Restating the code:**
  `i = i + 1   # increment i`
- **Explaining a bad name instead of fixing it:**
  `d = 30   # days until expiry` → `daysUntilExpiry = 30`
- **Commented-out code.** Delete it; version control keeps the history.
- **Change logs and author notes** (`# changed by Ana on 3 May`). Version control records these.
- **Closing-brace markers** (`} // end if`). If they're needed, the block is too long.
- **Section dividers inside functions.** If a function needs sections, consider splitting it.

Before writing a comment, ask whether a better name or a small extracted function would make it unnecessary. If yes, change the code. If the comment explains *why*, keep it.

## 3. Documenting public interfaces

Document anything other code depends on: exported functions, public classes, module interfaces, and API endpoints. Use the language's standard documentation format (docstrings, JSDoc, Javadoc, and so on).

Cover:

- What it does, in one sentence.
- Parameters, especially units, formats, and allowed ranges.
- What it returns, including what "nothing found" looks like.
- How it fails: errors raised or returned, and when.
- Side effects, if any.

Private helpers rarely need formal documentation if their names and signatures are clear.

## 4. TODOs

A TODO is a promise to someone in the future. Make it useful:

```
Poor:   # TODO: fix this
Better: # TODO(PAY-342): Handle partial refunds; currently refunds the full amount only.
```

- Say what is missing and why it matters.
- Reference a ticket where the project uses them.
- Don't leave TODOs for work that should be done in the current change.
- Never leave a TODO in place of handling an error or a security check.

## 5. Keeping comments true

A wrong comment is worse than no comment, because readers trust it.

- When you change code, read the comments around it and update any your change affects.
- When a comment no longer adds information, delete it.
- Keep comments next to the code they describe, so they're seen when the code changes.