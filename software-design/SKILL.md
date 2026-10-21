---
name: software-design
description: Principles for designing and structuring code so it stays simple, understandable, and easy to change, in any language or framework. Covers managing complexity, module boundaries and data ownership, layering, coupling and cohesion, domain modelling, SOLID, and avoiding over-engineering. Use this skill whenever writing a new feature, adding a module, file, class, or function, refactoring, deciding where code belongs, introducing an abstraction or interface, or reviewing the structure of existing code, even if the task does not mention "design" or "architecture".
---

# Software Design

The purpose of software design is to keep complexity under control. Complexity is anything that makes a system hard to understand or hard to change. It rarely arrives in one big decision; it accumulates through hundreds of small ones. Every rule in this skill exists to make those small decisions well.

Read the project's `AGENTS.md` before applying anything here. It describes the project's language, structure, and conventions. Apply these principles through the project's own patterns, and where the project sets a different convention, follow the project.

## Before writing code

Answer these questions first. They take a moment and prevent most structural mistakes.

1. **Who owns this?** Which module is responsible for this feature and the data it touches? If none fits, is this a new module, or does it belong to an existing one?
2. **Where does each piece go?** Separate the parts of the change: receiving input, business rules, storing or fetching data, and talking to external systems. Each belongs in its own layer.
3. **Does it cross a boundary?** If this code needs another module's data or behaviour, use that module's public interface. Never reach into its internals or its data storage.
4. **Is a new abstraction justified?** Only add an interface, base class, factory, or shared helper if there is a real, present need for it, not an imagined future one.
5. **Is there a decision record?** Check the project for Architecture Decision Records. Don't "fix" something that was decided deliberately.

## Core rules

### 1. Optimise for the reader

Code is read far more often than it is written. Prefer the design that a newcomer understands fastest, even if it is slightly longer. Obvious code beats clever code.

### 2. Build deep modules

A good module offers a simple interface and hides substantial functionality behind it. A shallow module, whose interface is nearly as complex as what it does, adds cost without adding value.

- Hide implementation details. Callers should know *what* a module does, not *how*.
- Don't split code into many tiny functions or classes that each do almost nothing. Splitting is only useful when each piece is meaningful on its own.
- Pull complexity downward: the module should handle hard cases internally rather than pushing them onto every caller.

For more depth, read `references/complexity.md`.

### 3. Divide by feature, then by layer

Organise code first by what it does for the business (orders, payments, users), then by technical layer inside each feature. Related code stays together, so most changes touch one place.

Within a feature, keep these responsibilities separate:

| Layer | Responsibility | Must not |
|---|---|---|
| Entry point (handler, controller, command, UI event) | Receive input, call business logic, return output | Contain business rules or data access |
| Business logic (service, use case, domain) | Apply the rules of the business | Know about HTTP, UI, or storage details |
| Data access (repository, gateway, client) | Read and write data or call external systems | Make business decisions |

Dependencies point inward: entry points depend on business logic; business logic depends on abstractions of data access, not on a specific database or framework.

### 4. Each module owns its data

A module is the only code that reads or writes its own data. Other modules ask it through its public interface. Expose that interface explicitly (for example, an index or public API file) and treat everything else in the module as private.

This single rule prevents most systems from turning into tangled code where every part depends on every other part.

For more depth, read `references/boundaries.md`.

### 5. Keep coupling low and cohesion high

- Things that change together belong together (high cohesion).
- Things that change independently should depend on each other as little as possible (low coupling).
- Talk only to direct collaborators. Avoid chains like `order.customer.account.address.city`, which tie your code to the internal structure of many other objects.
- Everything a module exposes will eventually be relied on by someone. Expose as little as possible.

For more depth, read `references/coupling.md`.

### 6. Make invalid states impossible

Design data types so that invalid combinations can't be represented.

- Use enumerations or specific types instead of free strings and loose flags.
- Convert raw input into trusted, typed values once, at the boundary. After that point, code can rely on the data being valid without checking again.
- Keep business rules in pure functions where possible, with storage, network, and time pushed to the edges. Pure logic is easy to test and reason about.
- Prefer immutable data. Data that can't change can't be changed incorrectly.

For more depth, read `references/domain-modelling.md`.

### 7. Prefer simplicity to flexibility

AI agents in particular tend to over-engineer. Actively resist it.

- **YAGNI:** don't build for requirements that don't exist yet.
- **No speculative abstractions:** an interface with exactly one implementation and no realistic second one is usually noise. Exceptions: boundaries to external systems you need to replace in tests, or an interface the project's conventions require.
- **Rule of three:** tolerate duplication until the same logic appears a third time and the shared pattern is clearly understood. Duplication is cheaper to fix than the wrong abstraction.
- **Composition over inheritance:** combine small pieces rather than building deep class hierarchies.
- **Start with a modular monolith:** a single deployable application with well-separated modules. Split into separate services only when there is a concrete reason, such as independent scaling or independent teams.

### 8. Apply SOLID with judgement

The SOLID principles are guides, not laws. Apply them where they reduce complexity, and bend them where they would add it.

- **Single Responsibility:** a module has one reason to change.
- **Open/Closed:** add behaviour by adding code, rather than by editing stable code, where variation is real and expected.
- **Liskov Substitution:** a subtype must work anywhere its parent is expected.
- **Interface Segregation:** prefer small, focused interfaces.
- **Dependency Inversion:** business logic depends on abstractions; details plug in from outside.

For examples and guidance on when to bend each, read `references/solid.md`.

### 9. Wire things together in one place

Create and connect dependencies (database connections, clients, services) at the application's starting point, and pass them in. Business logic never constructs its own infrastructure. Handle cross-cutting concerns such as authentication, logging, and request tracking once, in a shared layer like middleware, not repeatedly inside each feature.

### 10. Let tools enforce the rules

Written rules are eventually broken. Where the project has architecture checks (such as dependency or import linting), run them after making changes and fix any violations. If none exist and the project would benefit, suggest adding them rather than adding them unasked.

For more depth, read `references/enforcement.md`.

## Example

A request to register a user, written two ways. Shown as pseudocode; apply the same shape in any language.

**Poor:** everything in the entry point.

```
handleRegister(request):
    if request.body.email does not contain "@": return 400
    existing = database.query("SELECT * FROM users WHERE email = ?", request.body.email)
    if existing: return 409
    hash = bcrypt(request.body.password)
    database.query("INSERT INTO users ...", request.body.email, hash)
    emailClient.send(request.body.email, "Welcome!")
    return 201
```

The handler mixes input parsing, business rules, storage, and messaging. None of it can be reused or tested without HTTP and a real database, and every change touches this one function.

**Better:** each responsibility in its own place.

```
# Entry point: input in, output out
handleRegister(request):
    input = parseRegistration(request.body)        # raises a validation error if invalid
    user = userService.register(input)
    return created(toUserResponse(user))

# Business logic: the rules, no HTTP, no SQL
userService.register(input):
    if userRepository.existsByEmail(input.email):
        raise EmailAlreadyRegistered
    user = User.create(input.email, passwordHasher.hash(input.password))
    userRepository.save(user)
    notifications.sendWelcome(user)
    return user

# Data access: storage only
userRepository.existsByEmail(email): ...
userRepository.save(user): ...
```

Each piece can now be understood, tested, and changed on its own. Note what was *not* added: no factory, no abstract base service, no generic repository framework. The structure is as simple as the problem allows.

## Common mistakes

- Putting business rules in handlers, controllers, or UI components.
- Querying another module's tables or importing its private files.
- Creating an interface, base class, or "manager" for a single use.
- Extracting shared helpers after seeing duplication only twice, then bending the helper with flags as cases diverge.
- Breaking a clear 30-line function into ten 3-line functions that must be read together.
- Passing raw, unvalidated input deep into the system and re-checking it in many places.
- Representing states with loose strings or combinations of booleans.
- Constructing database or network clients inside business logic.
- Proposing microservices when well-separated modules would do.

## Reference files

Open these only when the task calls for more depth.

| File | Read when |
|---|---|
| `references/complexity.md` | Deciding how to split or combine code, or whether an abstraction is worth it |
| `references/boundaries.md` | Creating a module, deciding where code lives, or connecting modules |
| `references/coupling.md` | Changes in one place keep breaking another, or designing a public interface |
| `references/domain-modelling.md` | Designing data types, validation, or business rules |
| `references/solid.md` | Designing classes and interfaces, or reviewing object-oriented code |
| `references/enforcement.md` | Setting up or running automated architecture checks |

## Sources

John Ousterhout, *A Philosophy of Software Design*; Robert C. Martin, *Clean Architecture*; Edsger Dijkstra, "On the role of scientific thought" (separation of concerns); David Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules"; Eric Evans, *Domain-Driven Design*; Sandi Metz, "The Wrong Abstraction"; Alistair Cockburn, Hexagonal Architecture; Gary Bernhardt, "Boundaries"; Yaron Minsky, "Effective ML"; Alexis King, "Parse, don't validate".