# Boundaries, Modules, and Layers

This file explains how to divide a system into parts and how those parts should connect.

## Contents

1. Modules by business capability
2. Layers inside a module
3. The dependency rule
4. Public interfaces and data ownership
5. Communication between modules
6. Cross-cutting concerns
7. The composition root
8. Modular monolith first

## 1. Modules by business capability

Divide the system by what it does for the business, not by technical role.

```
Preferred: by feature                Avoid: by technical role
modules/                             controllers/
  orders/                              orders, users, payments...
    (entry, logic, data)               
  payments/                          services/
    (entry, logic, data)               orders, users, payments...
  users/                             repositories/
    (entry, logic, data)               orders, users, payments...
```

With feature-based modules, a change to orders stays inside `orders/`. With role-based folders, the same change is spread across the whole codebase.

Each module should correspond to a **bounded context**: an area with its own vocabulary and rules. The same word can mean different things in different modules. A "customer" in billing (payment details, invoices) and a "customer" in support (tickets, history) can be separate models. Don't force one shared model on both.

## 2. Layers inside a module

Within each module, separate:

- **Entry points:** HTTP handlers, controllers, CLI commands, message consumers, UI event handlers. They translate the outside world into calls to business logic and translate results back.
- **Business logic:** the rules. Services, use cases, domain objects. This is the part that should be most stable and most tested.
- **Data access and integrations:** repositories, database queries, API clients. They translate between business objects and storage or external systems.
- **Contracts:** the module's types, input schemas, and public interface.

A small module may not need every layer as a separate file. Add structure as the module grows; don't create empty layers in advance.

## 3. The dependency rule

Dependencies point toward business logic, never away from it.

```
Entry points  ──►  Business logic  ◄──  Data access
```

- Business logic does not import web frameworks, HTTP types, database drivers, or UI libraries.
- Business logic defines what it needs (for example, "something that can save a user") and data access provides it.
- This means the framework and database are details. They can be changed, and business logic can be tested without them.

This is the core idea shared by Clean Architecture, Hexagonal Architecture (Ports and Adapters), and Onion Architecture.

Apply the rule pragmatically. In a small application, a service calling a repository directly is fine, as long as the service doesn't contain storage details such as query strings.

## 4. Public interfaces and data ownership

Each module exposes one explicit public interface, such as an index file, a public package, or an exported API, and keeps everything else private.

Rules:

- Other modules import only from the public interface.
- Only the owning module reads or writes its data. No other module queries its tables, collections, or files.
- The public interface returns the module's own types or plain data, not internal objects that expose its storage details.

Why this matters: when every module can reach into every other module's data, any change to a table can break code anywhere. With ownership, a module can change its storage freely as long as its interface holds.

## 5. Communication between modules

Prefer, in order:

1. **Direct calls through the public interface**, for simple, synchronous needs.
2. **Events**, when one module needs to react to something in another without the first knowing about it. For example, `orders` announces "order placed"; `notifications` and `analytics` react. The publisher doesn't depend on the listeners.

Avoid:

- Circular dependencies between modules. If A needs B and B needs A, the boundary is probably wrong: merge them, or extract the shared concept into a third module.
- Shared "common" or "utils" modules that collect business logic. Keep shared code limited to genuinely generic utilities with no business rules.

## 6. Cross-cutting concerns

Some concerns apply everywhere: authentication, authorisation checks at the request level, logging, request IDs, rate limiting, error-to-response mapping, transactions per request.

Handle each in one place, typically middleware, interceptors, decorators, or a wrapper around entry points. Feature code should not repeat this logic. A handler that starts with ten lines of authentication and logging setup is a sign the concern belongs in shared infrastructure.

## 7. The composition root

Create and connect all dependencies in one place, at application start:

```
main():
    config = loadConfig()
    database = connectDatabase(config.databaseUrl)
    userRepository = UserRepository(database)
    userService = UserService(userRepository, PasswordHasher())
    server = createServer(routes(userService))
    server.start()
```

Business logic receives its dependencies; it never creates them. This makes dependencies visible and makes testing straightforward, because tests pass in substitutes.

A dependency injection framework is optional. Passing dependencies through constructors or function parameters is often clearer.

## 8. Modular monolith first

Start as a single deployable application with strongly separated modules. This gives clear boundaries, simple deployment, simple debugging, and transactions that just work.

Consider splitting a module into a separate service only when there is a concrete need, such as:

- It must scale very differently from the rest.
- A separate team must deploy it independently.
- It has very different reliability, security, or technology requirements.

Well-defined module boundaries make this split straightforward later. Poorly defined boundaries make microservices a distributed version of the same tangle, with network failures added.