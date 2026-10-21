# Coupling and Cohesion

Coupling is how much one piece of code depends on another. Cohesion is how closely the parts of one piece of code belong together. Aim for low coupling between modules and high cohesion within them.

## Contents

1. Cohesion
2. Kinds of coupling (connascence)
3. Law of Demeter
4. Hyrum's Law and interface design
5. Temporal coupling
6. Practical checks

## 1. Cohesion

A cohesive module's contents serve one purpose and change for the same reasons.

Signs of low cohesion:

- A module named `utils`, `helpers`, `common`, or `misc` that keeps growing.
- A class whose methods use entirely different subsets of its fields.
- A file that changes for many unrelated reasons.

Fix low cohesion by moving each piece to the module whose purpose it serves.

## 2. Kinds of coupling (connascence)

Connascence, a term from Meilir Page-Jones, names the ways two pieces of code must agree for the system to work. Weaker forms are cheaper. Listed from weakest to strongest:

| Kind | The two pieces must agree on | Example |
|---|---|---|
| Name | A name | A function name; renaming requires updating callers |
| Type | A type | A parameter must be a `User` |
| Meaning | The meaning of a value | `status = 2` means "shipped" |
| Position | Order of values | `createUser("Ana", "ana@x.com", true, false)` |
| Algorithm | An algorithm | Both sides must hash passwords the same way |
| Execution order | The order of calls | `init()` must run before `process()` |
| Timing | When things happen | Two processes rely on a delay |
| Value | Values that must change together | Two constants that must stay in sync |
| Identity | The same instance | Two components must share one object |

Rules:

- Convert strong forms into weak ones. Replace magic values with named constants or enumerations (meaning → name). Replace long positional argument lists with named parameters or an object (position → name). Put shared algorithms in one place (algorithm → name).
- Strong forms of coupling are acceptable inside a module, where the code is close together. Across module boundaries, keep to the weakest forms possible.
- The farther apart two pieces of code are, the weaker their coupling should be.

## 3. Law of Demeter

A function should talk only to its direct collaborators: its own object, its parameters, objects it creates, and its direct fields.

```
Avoid:   order.getCustomer().getAccount().getAddress().getCity()
Prefer:  order.shippingCity()
```

The first version depends on the internal structure of four objects. If any of them changes, this code breaks. The second asks the object for what is needed and lets it handle the details.

This applies to object-oriented chains, not to fluent builders or data transformations such as `list.filter(...).map(...)`, which operate on one value.

## 4. Hyrum's Law and interface design

With enough users of an interface, every observable behaviour will be depended on by someone, including behaviour you never promised: ordering of results, error message text, timing, extra fields.

Implications:

- Expose the minimum. Every exported function, field, and type is a commitment.
- Return only what callers need, not entire internal objects.
- Be deliberate about defaults, ordering, and error formats, because callers will rely on them.
- When changing behaviour that others may depend on, treat it as a breaking change.

## 5. Temporal coupling

Code is temporally coupled when operations must happen in a particular order but nothing enforces it.

```
Avoid:
    report = Report()
    report.setData(data)     # forgetting this causes a crash later
    report.generate()

Prefer:
    report = Report(data)    # cannot exist without its data
    report.generate()
```

Make required order impossible to get wrong: require dependencies at construction, return a new object from each step, or combine steps that must always happen together into one operation.

## 6. Practical checks

Ask of any change:

- If I change this module's internals, which other modules could break? The answer should be none.
- Does this module import from another module's private files?
- Does this function know about the internal structure of objects it didn't create?
- Are there values that must stay in sync across files with nothing enforcing it?
- Am I exposing anything callers don't need?