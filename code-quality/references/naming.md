# Naming

A good name tells the reader what something is or does, without making them read its implementation. Naming is the cheapest and most effective way to make code understandable.

## Contents

1. Principles
2. Naming by kind
3. Words to avoid
4. Consistency
5. Scope and length
6. Renaming

## 1. Principles

- **Reveal intent.** `daysUntilExpiry` says what it holds; `d` and `days` don't say enough.
- **Be precise.** `fetchActiveUsers` is better than `getUsers` if it only returns active users and calls the network.
- **Avoid misleading names.** Don't call something `userList` if it's a set or a map. Don't call a function `isValid` if it also corrects the data.
- **Make differences meaningful.** `account` and `accountData` or `user1` and `user2` force readers to guess the difference. Name what actually differs: `sourceAccount`, `targetAccount`.
- **Use pronounceable, searchable names.** `generationTimestamp`, not `genymdhms`. A name that can be searched for is easier to trace than a single letter or a number.
- **Name the concept, not the type.** `customers`, not `customerArray`. The type may change; the concept won't.

## 2. Naming by kind

| Kind | Form | Examples |
|---|---|---|
| Modules, classes, types | Noun or noun phrase, from the business domain | `Invoice`, `PaymentGateway`, `ShippingAddress` |
| Functions performing actions | Verb or verb phrase | `sendInvoice`, `cancelSubscription` |
| Functions returning values | Noun, or `get`/`find`/`calculate` + noun | `totalPrice()`, `findUserByEmail` |
| Booleans | A question answered yes or no | `isExpired`, `hasPermission`, `canEdit`, `shouldRetry` |
| Collections | Plural noun | `orders`, `pendingPayments` |
| Maps / dictionaries | Describe the mapping | `priceByProductId`, `usersByEmail` |
| Constants | The meaning of the value | `MAX_UPLOAD_SIZE_BYTES`, `DEFAULT_PAGE_SIZE` |
| Event names | Past tense | `OrderPlaced`, `PaymentFailed` |

Conventions for verbs, applied consistently:

- `get` returns something quickly and simply, usually already in memory.
- `find` or `lookup` searches and may return nothing.
- `fetch` or `load` retrieves from elsewhere, such as a network or database.
- `create` makes something new; `build` assembles something from parts.
- `calculate` or `compute` derives a value.
- `ensure` makes a condition true, creating or fixing as needed.

Follow the project's existing usage where it differs.

## 3. Words to avoid

These words usually hide what something does. Replace them with the specific responsibility.

| Vague | Ask | Possible replacement |
|---|---|---|
| `Manager`, `Handler`, `Processor` | What does it manage or handle? | `InvoiceSender`, `RetryScheduler` |
| `Helper`, `Util`, `Common` | What is it about? | Move functions to the module they serve |
| `data`, `info`, `object`, `value` | Data about what? | `userProfile`, `shippingQuote` |
| `do`, `handle`, `process`, `perform` | Do what? | `validate`, `charge`, `archive` |
| `temp`, `tmp`, `foo`, `x` | What is it temporarily? | `unsavedDraft` |
| `flag`, `status` (alone) | Flag for what? Status of what? | `isArchived`, `paymentStatus` |
| `new`, `old`, `2`, `final` | Different how? | Name the actual difference |

Exceptions: framework-required names, and terms the project already uses consistently, such as a "handler" layer.

## 4. Consistency

- One word per concept. If the codebase says `customer`, don't introduce `client` or `user` for the same thing.
- One concept per word. Don't use `add` to mean both "insert into a list" and "sum two numbers".
- Match the project's casing conventions (camelCase, snake_case, PascalCase) for each kind of name.
- Mirror names across layers: an `Invoice` type, `InvoiceRepository`, `invoice` table, and `/invoices` endpoint are easy to connect.

## 5. Scope and length

The wider a name's scope, the more descriptive it needs to be.

- A loop index in three lines: `i` is fine.
- A local variable in a short function: a short but clear name, such as `total`.
- A module-level constant or public function: fully descriptive, such as `calculateMonthlyRecurringRevenue`.

Don't encode types or scopes into names (`strName`, `m_count`, `IUserService`) unless the project's conventions require it.

## 6. Renaming

- Rename when a name is misleading or the concept has changed. A wrong name costs every reader.
- Use the editor's or language tools' rename feature so all references change together.
- Renaming widely used code is a separate, focused change, not something to slip into a feature change.
- Renaming public interfaces (APIs, exported functions, database columns) can break other code. Treat it as a breaking change.