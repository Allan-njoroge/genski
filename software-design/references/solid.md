# SOLID, Applied with Judgement

The SOLID principles, collected by Robert C. Martin, help keep modules focused and replaceable. Each is useful when it reduces complexity and harmful when applied mechanically. For each principle, this file gives the idea, an example, and when to bend it.

## Contents

1. Single Responsibility
2. Open/Closed
3. Liskov Substitution
4. Interface Segregation
5. Dependency Inversion
6. Composition over inheritance

## 1. Single Responsibility Principle

**Idea:** a module should have one reason to change, meaning it serves one actor or one area of concern.

```
Avoid:
    Invoice
        calculateTotal()      # changes when pricing rules change (finance)
        renderPdf()           # changes when layout changes (design)
        saveToDatabase()      # changes when storage changes (engineering)

Prefer:
    Invoice.calculateTotal()
    InvoicePdfRenderer.render(invoice)
    InvoiceRepository.save(invoice)
```

**When to bend it:** "one reason to change" is not "one function per class" or "one tiny thing per file". Splitting closely related logic into many fragments creates shallow modules. Group code that changes together, even if it does several steps.

## 2. Open/Closed Principle

**Idea:** add new behaviour by adding code, rather than by editing stable, working code.

```
Avoid: a growing conditional edited for every new case
    calculateShipping(order):
        if order.method == "standard": ...
        else if order.method == "express": ...
        else if order.method == "drone": ...     # edit again for every new method

Prefer, when new methods are genuinely expected:
    ShippingMethod.calculate(order)
    StandardShipping, ExpressShipping, DroneShipping each implement it
```

**When to bend it:** a small, stable conditional is clearer than a family of classes. Introduce the extension point when variation actually shows up (often the third case), not in anticipation.

## 3. Liskov Substitution Principle

**Idea:** any subtype must work correctly wherever its parent type is expected, without callers needing to know which one they have.

```
Violation:
    ReadOnlyRepository extends Repository
        save(item): throw "Not supported"     # callers of Repository now crash
```

Signs of a violation: overrides that throw "not supported", subtypes that ignore inputs, or callers that check the concrete type before calling.

**Fix:** split the type so the subtype only claims what it can do (for example, separate `Reader` and `Writer` interfaces), or use composition instead of inheritance.

## 4. Interface Segregation Principle

**Idea:** callers should not depend on methods they don't use. Prefer several small, focused interfaces over one large one.

```
Avoid:
    UserStore: find, save, delete, exportCsv, sendNewsletter, rebuildIndex

Prefer:
    UserReader: find
    UserWriter: save, delete
```

A module that only reads users then depends only on `UserReader`, and is unaffected by changes to writing or exporting.

**When to bend it:** don't split an interface that all its callers use whole. Many one-method interfaces with a single implementation are noise.

## 5. Dependency Inversion Principle

**Idea:** business logic should not depend on details such as databases, frameworks, or third-party services. Both should depend on an abstraction that business logic defines.

```
Avoid:
    OrderService
        db = PostgresClient(...)         # business logic tied to one database
        stripe = StripeSdk(...)          # and one payment provider

Prefer:
    OrderService(orderRepository, paymentGateway)
    # The service declares what it needs.
    # The composition root passes in concrete implementations.
```

Benefits: business logic can be tested with simple substitutes, and infrastructure can change without touching business rules.

**When to bend it:** you don't need an abstraction for every dependency. Invert dependencies on things that are external, slow, volatile, or need substituting in tests: databases, network services, the clock, the file system. Stable, in-process libraries can be used directly.

## 6. Composition over inheritance

Not part of SOLID, but closely related and from the same tradition (the "Gang of Four" *Design Patterns* book).

Inheritance creates the strongest possible coupling: the child depends on the parent's internals, and changes to the parent ripple to every child. Deep hierarchies become rigid and hard to understand.

Prefer building behaviour by combining objects:

```
Avoid:
    Animal → Bird → FlyingBird → SwimmingFlyingBird → Duck

Prefer:
    Duck has movement abilities: [Flying, Swimming, Walking]
```

Use inheritance only for a true "is a" relationship where the subtype can fully substitute for the parent, and keep hierarchies shallow, ideally one level.