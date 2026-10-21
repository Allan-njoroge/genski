# Enforcing Architecture Automatically

Design rules written only in documentation are gradually broken, by people and by agents, usually without anyone noticing until the structure has decayed. Automated checks catch violations the moment they happen.

## Contents

1. Fitness functions
2. What to check
3. Tools by ecosystem
4. How to work with checks as an agent
5. Introducing checks to a project

## 1. Fitness functions

A fitness function is an automated test of an architectural property, rather than of behaviour. The term comes from *Building Evolutionary Architectures* by Neal Ford, Rebecca Parsons, and Patrick Kua. Like unit tests, fitness functions run in continuous integration and fail the build when a rule is broken.

## 2. What to check

The most valuable rules to enforce, in rough order:

1. **Module boundaries:** modules import other modules only through their public interfaces.
2. **Layer direction:** entry points may depend on business logic; business logic must not depend on entry points, web frameworks, or database drivers.
3. **No circular dependencies** between modules.
4. **Data ownership:** only a module's own data-access code touches its tables or collections. This is harder to check automatically; naming conventions plus import rules usually cover it.
5. **Forbidden dependencies:** for example, no direct use of a deprecated library, or no imports of test code from production code.

Other useful checks include limits on file or function complexity, and checks that public APIs haven't changed without a version change.

## 3. Tools by ecosystem

These are commonly used options. Use what the project already has, and check the project's `AGENTS.md` for the command to run.

| Ecosystem | Tools |
|---|---|
| JavaScript / TypeScript | dependency-cruiser, eslint-plugin-boundaries, ESLint `no-restricted-imports`, madge (circular dependencies) |
| Python | import-linter, pydeps, Ruff rules |
| Java / Kotlin | ArchUnit, Konsist (Kotlin), Spring Modulith (Spring) |
| .NET | NetArchTest, ArchUnitNET |
| Go | go-arch-lint, depguard (via golangci-lint) |
| Any | A small custom test that scans import statements |

## 4. How to work with checks as an agent

- After structural changes, run the architecture check command from `AGENTS.md`, along with tests and linting.
- When a check fails, fix the design, not the rule. Move the code to the right module, go through the public interface, or invert the dependency.
- Never weaken, disable, or add exceptions to an architecture rule to make a change pass. If you believe a rule is wrong for this case, stop and explain why to the person you are working with, and let them decide.

## 5. Introducing checks to a project

If a project has clear module boundaries but no checks, suggesting them is valuable. Recommend, don't impose:

1. Start with one or two rules that the codebase already follows, such as no circular dependencies and no imports of other modules' private files.
2. Run them in continuous integration so violations block merging.
3. If existing code violates a rule, record the existing violations as a known baseline and prevent new ones, then fix the baseline over time.
4. Add rules gradually as the team agrees on them.