# Managing Complexity

Complexity is anything about a system's structure that makes it hard to understand or change. This file explains how to recognise it and how to keep it low.

## Contents

1. Symptoms of complexity
2. Deep and shallow modules
3. Information hiding
4. Pulling complexity downward
5. When to split and when to combine
6. Abstractions: when they help and when they hurt
7. Duplication and the rule of three

## 1. Symptoms of complexity

Watch for these signs in code you write or review:

- **Change amplification:** a simple change requires edits in many places.
- **Cognitive load:** a reader must hold many details in mind to make a change safely.
- **Unknown unknowns:** it is unclear which code must change, or what a change might break. This is the most dangerous symptom.

Complexity comes from two sources: **dependencies** (code that can't be understood or changed in isolation) and **obscurity** (important information that isn't obvious).

## 2. Deep and shallow modules

Picture a module as a rectangle: the top edge is its interface (what callers must know), the area is its functionality (what it does for them).

- A **deep module** has a small interface and large functionality. A file system's `open`, `read`, `write`, `close` hide disk layout, caching, permissions, and scheduling. Callers gain a lot and learn little.
- A **shallow module** has an interface nearly as complex as its implementation. It adds a layer callers must learn without hiding anything meaningful.

Signs of a shallow module:

- A class whose methods each forward a call to another object.
- A function whose name and parameters fully describe its body.
- A "manager", "helper", or "util" that wraps one call.

Shallow modules are sometimes unavoidable, but every one adds cost. Prefer fewer, deeper modules.

## 3. Information hiding

Each module should encapsulate decisions that callers don't need to know: data formats, storage choices, algorithms, third-party libraries.

- If two modules both know the same detail (such as a file format), changing that detail means changing both. This is **information leakage**. Move the knowledge into one module.
- Don't structure code by the order things happen (read, then process, then write) if each step needs the same knowledge. That spreads the knowledge across steps. Structure by the knowledge each part owns.
- Private by default. Make something public only when a caller genuinely needs it.

## 4. Pulling complexity downward

When complexity is unavoidable, the module should absorb it, not its callers. A module has one implementation but many callers, so handling a difficulty once inside is far cheaper than handling it everywhere outside.

- Choose sensible defaults so most callers pass few options.
- Handle common error cases internally where the module can recover meaningfully.
- Don't expose configuration parameters just to avoid making a decision. Each parameter pushes a decision onto every caller.

## 5. When to split and when to combine

Split code apart when:

- The pieces are unrelated and each is understandable alone.
- One piece is general-purpose and the other specific to a use.
- A function does several distinct things that are each meaningful.

Keep code together when:

- The pieces share information or must be read together to be understood.
- Splitting would create a shallow function or class.
- Combining simplifies the interface or removes duplication.

A long function is acceptable if it reads clearly from top to bottom as one coherent task. Length alone is not a reason to split. Splitting into fragments that jump back and forth harms readability more than length does.

## 6. Abstractions: when they help and when they hurt

An abstraction is worth adding when:

- It hides a genuinely complex or volatile detail (a payment provider, a storage engine).
- There are two or more real implementations now, or a test double that needs to replace a slow or external dependency.
- It names a concept the business actually uses.

An abstraction is harmful when:

- It exists for a future that may never arrive.
- It has one implementation and nothing to hide.
- Callers must understand the implementation anyway to use it correctly.
- It is a generic framework (a "base repository", "abstract service factory") built before the specific cases are understood.

When unsure, write the concrete code. Extracting an abstraction later from working code is easy. Removing a wrong abstraction that everything depends on is hard.

## 7. Duplication and the rule of three

The DRY principle ("Don't Repeat Yourself") is about knowledge, not text. Two pieces of code that look alike but represent different business rules are not duplicates; they will change for different reasons.

- First occurrence: write it.
- Second occurrence: note the similarity, but duplicate it.
- Third occurrence: if the pattern is now clear and the cases truly share one rule, extract it.

If an extracted helper starts gaining flags and conditionals to serve diverging callers, it was the wrong abstraction. Inline it back into the callers and let each evolve on its own.