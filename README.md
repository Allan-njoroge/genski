# genski

**General skills for AI coding agents.**

genski is a library of engineering knowledge written for AI coding agents. It teaches agents such as Claude Code, Codex, and Gemini CLI how experienced software engineers think, decide, and work, so that every piece of code an agent writes follows the same high standards, in any language, framework, or project.

The name is short for *general skills*.

---

## What genski is

Every experienced engineer carries a set of habits built over years: how to divide a system into parts, when to add an abstraction and when to hold back, how to handle failure, how to write a test that stays useful, how to keep an application secure. These habits are what separate code that keeps working and stays easy to change from code that slowly becomes painful to touch.

genski writes those habits down in a form agents can read and follow.

It draws on the principles that the most respected engineers and organisations in the industry agree on, including the work of John Ousterhout, Robert C. Martin, Martin Fowler, Kent Beck, Eric Evans, Michael Nygard, Edsger Dijkstra, and David Parnas, along with established standards such as the OWASP Top 10 and the Twelve-Factor App. genski turns these ideas into concrete rules, checklists, and examples an agent can apply to the task in front of it.

Each piece of knowledge in genski is a **skill**: a folder containing a `SKILL.md` file with focused instructions on one area of engineering, such as software design, testing, or security. Together, the skills form a complete guide to building quality software.

---

## What genski does

### It gives agents engineering judgement

On their own, AI agents write code that works but often lacks the judgement of an experienced engineer. They tend to add layers and abstractions nobody needs, place logic in the wrong part of a system, skip error handling, or write tests tied so closely to the implementation that they break with every change.

genski supplies that judgement. Before an agent writes code, the relevant skills guide it through the same questions a senior engineer would ask:

- Which part of the system owns this feature and its data?
- Which layer does each piece of this code belong in?
- Is this new abstraction justified by a real need today?
- What happens if this operation fails halfway, or runs twice?
- How will someone test this, change it, or debug it in six months?

### It keeps quality consistent

Every agent using genski follows the same rules. Code written by Claude Code on one day and by Codex the next follows the same design principles, the same testing approach, and the same security practices. The same holds across team members: everyone's agents share one standard.

### It works across languages and frameworks

genski describes principles, not tools. Its rules hold whether a project is written in TypeScript, Python, Go, or Java, and whether it is a web API, a frontend application, or a command-line tool. One library serves every project.

### It adapts to each project

Each project describes its own stack, structure, and conventions in an `AGENTS.md` file at its root. genski's skills direct the agent to read that file first, then apply genski's principles through the project's own tools and patterns.

For example, genski teaches that a request handler should only read the request, call one piece of business logic, and return a response. A project's `AGENTS.md` then tells the agent where its handlers live and what they're called. The agent combines the two: the principle from genski, the specifics from the project.

Projects can also set their own conventions in `AGENTS.md`, and the agent follows the project's choice wherever the two differ. genski provides strong defaults; each project has the final say.

### It loads only what each task needs

genski is designed so agents can carry a lot of knowledge without being overwhelmed by it. It works in three stages:

1. **Discovery.** When an agent starts, it reads only the short name and description of each skill.
2. **Activation.** When a task matches a skill's description, the agent reads that skill's full `SKILL.md`, which holds the essential rules and checklist.
3. **Depth on demand.** Each `SKILL.md` points to more detailed reference files and says when to open them. The agent reads those only when the task calls for that depth.

A task to rename a variable loads almost nothing. A task to design a new feature loads the design, testing, and security guidance it needs. Either way, the agent's attention stays on the work.

---

## The skills

genski's skills are organised in two layers.

### Core skills

Core skills hold the principles true for all software, whether frontend, backend, or tooling. They shape nearly every line an agent writes.

**`software-design`** teaches agents to manage complexity, which is the central challenge of software. It covers dividing systems into modules with clear boundaries and clear data ownership, keeping modules loosely connected, building modules with simple interfaces that hide rich functionality, designing data types so invalid states can't exist, separating pure business logic from side effects, applying the SOLID principles, and resisting unnecessary abstraction.

**`code-quality`** teaches agents to write code that is easy to read and change: clear names, focused functions, predictable behaviour, and code that a newcomer can follow. It also covers recognising problems in existing code and improving it in small, safe steps.

**`error-handling`** teaches agents to detect problems early, stop clearly, and handle failures in the right place. It separates business errors, such as an insufficient balance, from technical errors, such as a lost database connection, and translates each into the right response at the edge of the system.

**`testing`** teaches agents to write tests that give real confidence: many fast, focused tests and fewer slow, broad ones; tests that are independent and repeatable; a consistent structure in every test; and tests that check behaviour, so code can be improved without rewriting them.

**`security`** teaches agents to treat every input as untrusted, check permissions on every action, grant each part of a system only the access it needs, keep secrets out of code, and build several layers of protection so no single mistake becomes a breach.

**`code-review`** teaches agents to review code the way a careful senior engineer would, checking for correctness, clarity, design, security, and test coverage, and giving feedback that improves the overall health of the codebase.

**`git-workflow`** teaches agents to make small, focused commits with clear messages, and to prepare changes that are easy for others to review.

**`decisions`** teaches agents to identify what matters most for a system, such as performance, security, or ease of change, before designing it; to treat hard-to-reverse decisions with extra care; and to record significant decisions so future readers, human or agent, understand why things are the way they are.

### Domain skills

Domain skills hold practices specific to one kind of software. They build on the core skills and add what is particular to their area.

**`backend-api-design`** teaches agents to design APIs that are predictable and safe to use: resources and methods that follow HTTP conventions, correct status codes, consistent error responses, pagination on lists, operations that are safe to retry, and versioning that protects existing clients.

**`backend-data`** teaches agents to work with data safely: grouping operations that must succeed or fail together, evolving database schemas without downtime, keeping data and messages consistent with each other, and writing queries that stay fast as data grows.

**`backend-reliability`** teaches agents to build services that stay up when things around them fail: timeouts on every external call, careful retries, stopping calls to services that are down, and keeping core features working when secondary ones fail.

**`backend-observability`** teaches agents to build systems that explain themselves: structured logs, meaningful metrics, and request tracing, so problems can be found and understood quickly in production.

**`frontend-components`** teaches agents to build interface components that are focused, reusable, and predictable, with clear handling of state and rendering.

**`frontend-accessibility`** teaches agents to build interfaces that everyone can use, including people who navigate by keyboard or use screen readers, following the WCAG guidelines.

---

## How a skill is structured

```
software-design/
├── SKILL.md              # essential rules, checklist, and examples
└── references/           # in-depth material, opened when a task needs it
    ├── complexity.md
    ├── boundaries.md
    └── domain-modelling.md
```

Every `SKILL.md` follows the same pattern:

- **A description** that tells the agent what the skill covers and when to use it.
- **A pre-work checklist** the agent runs through before writing code.
- **Concrete rules** written as specific instructions, so they can be applied directly.
- **Examples** showing good and poor approaches side by side.
- **Common mistakes** that agents and people tend to make in this area.
- **Pointers to references** for deeper material, with guidance on when each is needed.

Skills are written in the open [Agent Skills](https://agentskills.io) format, which is supported by Claude Code, Codex, Gemini CLI, Cursor, GitHub Copilot, and many other agents. One library serves every agent on the team.

---

## The foundations

genski's principles come from these works, among others. Each skill credits its sources so readers can go deeper.

- *A Philosophy of Software Design*, John Ousterhout
- *Clean Architecture* and *Clean Code*, Robert C. Martin
- *Refactoring*, Martin Fowler
- *Test-Driven Development: By Example*, Kent Beck
- *The Pragmatic Programmer*, Andrew Hunt and David Thomas
- *Domain-Driven Design*, Eric Evans
- *Release It!*, Michael Nygard
- *Building Evolutionary Architectures*, Neal Ford, Rebecca Parsons, and Patrick Kua
- [The Twelve-Factor App](https://12factor.net)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)