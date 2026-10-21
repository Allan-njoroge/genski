# Authorisation and Access Control

Authorisation decides what an authenticated user or system may do. Broken access control is the most common serious vulnerability in web applications and APIs, and it is almost always caused by a check that is missing rather than one that is wrong.

## Contents

1. Principles
2. Object-level authorisation
3. Function-level authorisation
4. Field-level authorisation
5. Multi-tenant systems
6. Where to put authorisation logic
7. Testing access control

## 1. Principles

- **Deny by default.** If there's no explicit rule allowing an action, refuse it.
- **Check on every request,** on the server, for every operation. Never rely on a previous check, a hidden button, or an unguessable URL.
- **Derive identity from the verified session or token,** never from request data.
- **Least privilege:** grant the minimum permissions needed for each role.
- **Fail closed:** if the authorisation check itself errors, refuse access.

## 2. Object-level authorisation

The most common access control flaw: a user changes an ID in a request and gets someone else's data. (OWASP calls this Broken Object Level Authorisation, also known as Insecure Direct Object Reference.)

```
Vulnerable:
    getOrder(request):
        return orders.find(request.params.orderId)       # any signed-in user can read any order

Safe:
    getOrder(request):
        order = orders.findForCustomer(request.params.orderId, customerId=request.user.id)
        if order is absent: raise NotFound
        return order
```

Rules:

- Every operation that takes an identifier checks that the current user may act on that specific object.
- Prefer **scoping queries** to the user (`WHERE id = $1 AND owner_id = $2`) over fetching first and checking after. It's harder to forget, and it can't leak data through a mistake later in the function.
- Return "not found" rather than "forbidden" when revealing that the object exists would itself leak information.
- Apply this to *every* way of reaching an object: reads, updates, deletes, exports, file downloads, nested resources (`/users/5/orders/9`), and bulk operations.
- Unguessable IDs (such as UUIDs) make attacks harder but are not a substitute for authorisation checks.

## 3. Function-level authorisation

Some operations are restricted to certain roles: administration, approvals, refunds, user management.

- Check the role or permission on the server for every restricted operation.
- Don't assume an endpoint is safe because it isn't linked in the interface. Attackers find unlinked endpoints.
- Keep administrative functions clearly separated, and apply the strictest checks to them.
- Prefer permissions ("can issue refunds") over checking role names throughout the code, so roles can change without editing every check.

## 4. Field-level authorisation

- **Reading:** return only the fields this user may see. Build explicit response objects rather than returning whole records, which can expose fields such as password hashes, internal notes, or other users' details.
- **Writing:** accept only the fields this user may change. Explicitly list allowed fields for each operation, so users can't set `role`, `isVerified`, `balance`, or `ownerId` by adding them to a request (mass assignment).

## 5. Multi-tenant systems

When one system serves several organisations (tenants):

- Every query on tenant data must be filtered by tenant, without exception.
- Take the tenant from the verified session, never from the request.
- Enforce tenant filtering centrally where possible (for example, with a repository layer that always applies it, or database row-level security), so a single forgotten filter can't expose another organisation's data.
- Test cross-tenant access explicitly.

## 6. Where to put authorisation logic

- **Authentication** (is there a valid user?) belongs in shared middleware, applied to every route except explicitly public ones.
- **Role and permission checks** belong in a central policy layer or helper, called consistently from entry points or services.
- **Ownership and business rules** ("only the author can edit a draft") belong in business logic, so they apply no matter which entry point calls it: API, background job, or admin tool.
- Keep public routes explicitly marked as public, so a new route is protected by default.

## 7. Testing access control

Write tests for the denials, not just the successes. For each protected operation, test:

- No credentials → 401.
- Valid user, but not permitted → 403 (or 404).
- Valid user, another user's object → denied.
- Valid user, another tenant's object → denied.
- Attempts to set protected fields → rejected or ignored.
- Valid user, own object, permitted action → allowed.