---
name: security
description: Principles for writing secure software and avoiding the most common vulnerabilities, in any language or framework. Covers untrusted input, injection, authentication, sessions, authorisation and access control, secrets, cryptography, sensitive data, dependencies and supply chain, secure configuration, files and outgoing requests, resource limits, and security logging. Use this skill whenever code handles user input, authentication, permissions, passwords, tokens, secrets, personal or payment data, database queries, file uploads or paths, outgoing requests, third-party packages, configuration, or anything exposed to a network, and whenever reviewing code for security, even if the task does not mention security.
---

# Security

Most security breaches don't come from sophisticated attacks. They come from ordinary mistakes: an unchecked input, a missing permission check, a password in a log, an outdated package. This skill exists to stop those mistakes from being written in the first place. Security is not a feature added at the end; it is a property of every line that handles input, identity, or data.

Read the project's `AGENTS.md` first. Use the project's existing authentication, validation, and security libraries and patterns. Where the project's conventions differ from this skill in ways that weaken security, follow the project but point out the risk to the person you are working with.

## Before you start

1. **What's untrusted here?** Every input from outside the code: users, requests, files, other services, environment, databases written by others.
2. **Who may do this?** Which users or systems are allowed to perform this action, on which data?
3. **What's sensitive?** Passwords, tokens, keys, personal data, payment data, internal details.
4. **What's the worst misuse?** If someone tried to abuse this feature, what would they try? (Read another user's data, run commands, overload it, escalate privileges.)

## Core rules

### 1. Treat all input as untrusted

Anything from outside the code may be malicious: request bodies, query parameters, headers, cookies, file contents and names, uploaded files, messages from queues, responses from other services, and data from databases that other systems write to.

- Validate at the boundary, before use. Define what is allowed (types, formats, lengths, ranges, allowed values) and reject everything else. Allow-lists are safer than block-lists.
- Parse input into trusted types once (see `software-design`, "Parse, don't validate").
- Validate on the server. Client-side validation improves user experience but provides no security.
- Limit sizes: request bodies, uploads, strings, arrays, and page sizes.

### 2. Never build code from data

Injection happens when input is combined into something that gets executed or interpreted: SQL, shell commands, HTML, queries, templates, file paths.

- **Database queries:** always use parameterised queries or the query builder's binding. Never concatenate or interpolate input into query strings.
- **Shell commands:** avoid them. If unavoidable, pass arguments as a list to a function that doesn't invoke a shell, and allow-list the values.
- **HTML output:** let the template engine or UI framework escape output. Never insert untrusted content as raw HTML. If rich text is required, use a well-maintained sanitiser.
- **Never execute input:** no `eval`, dynamic code generation, or dynamic imports from user-controlled values.
- **Deserialisation:** never deserialise untrusted data with formats that can construct arbitrary objects. Use plain data formats such as JSON and validate the result.

For more depth, read `references/input-and-injection.md`.

### 3. Check permission on every action, for every object

Broken access control is the most common serious vulnerability in web applications.

- **Deny by default.** Every endpoint, operation, and resource requires an explicit permission rule; access is refused unless a rule allows it.
- **Check object ownership, not just login.** Being signed in doesn't mean a user may read order 4821. Check that *this* user may perform *this* action on *this* record, every time.
- **Enforce on the server.** Never rely on hiding buttons or links in the interface.
- **Never trust identity or role from the client.** Take the user's identity from the verified session or token, never from a request field such as `userId` or `role` in the body.
- **Centralise authorisation logic,** so rules are consistent and can't be forgotten in one place.

For more depth, read `references/authorization.md`.

### 4. Use proven authentication, never your own

- Use the project's established authentication library or identity provider. Don't invent login, token, or session schemes.
- Hash passwords with a slow, salted algorithm designed for passwords: Argon2id, scrypt, or bcrypt, with current recommended settings. Never store passwords in plain text, encrypt them reversibly, or hash them with fast hashes such as MD5 or SHA-256 alone.
- Give tokens and sessions an expiry, invalidate them on sign-out and password change, and store them securely (for browsers, cookies marked `HttpOnly`, `Secure`, and `SameSite`).
- Protect sign-in and recovery flows: rate-limit attempts, give the same response whether or not an account exists, and make reset tokens single-use and short-lived.

For more depth, read `references/authentication-and-sessions.md`.

### 5. Grant the least privilege

Every user, service, process, database account, API key, and token should have only the permissions it needs, for only as long as it needs them.

- Application database users shouldn't own the schema or have administrative rights.
- API keys and tokens should be scoped to specific actions and resources.
- Processes should not run as administrator or root.

### 6. Keep secrets out of code

- Never write passwords, API keys, tokens, private keys, or connection strings into code, configuration files committed to version control, tests, documentation, or examples.
- Read secrets from environment variables or a secrets manager, as the project does.
- Keep local secret files (such as `.env`) out of version control, and provide a template (such as `.env.example`) with placeholder values.
- If a secret is ever committed or exposed, treat it as compromised: tell the person you are working with that it must be rotated. Removing it from the code doesn't remove it from history.

For more depth, read `references/secrets-and-data.md`.

### 7. Don't invent cryptography

- Use well-established libraries and high-level functions for encryption, hashing, and signing. Never design your own algorithms or protocols.
- Use secure random generators for tokens, IDs that must be unguessable, and keys. Never use ordinary random functions for security purposes.
- Use TLS for all network traffic, and never disable certificate verification.
- Compare secrets and signatures with constant-time comparison functions.

### 8. Protect sensitive data

- Collect and keep only the personal data you need, and only for as long as needed.
- Never log secrets, passwords, tokens, full card numbers, or unnecessary personal data. Log identifiers instead.
- Don't return more data than the client needs. Select explicit fields instead of returning whole database records.
- Encrypt sensitive data in transit always, and at rest where it matters, as the project's policies require.
- Don't leak internal details in errors (see `error-handling`).

### 9. Choose dependencies carefully

Every package you add is code you're trusting with your system.

- Prefer the standard library and packages the project already uses.
- Before adding a package, check it is real, maintained, and widely used. Watch for names that imitate popular packages.
- Avoid packages published only days ago: new releases are where malicious code most often appears before it's detected.
- Pin versions and commit the lock file. Run the project's vulnerability audit and fix serious issues.

For more depth, read `references/dependencies-and-configuration.md`.

### 10. Secure by default

- Production configuration must disable debug modes, detailed error pages, and development tools.
- Restrict cross-origin access (CORS) to known origins. Never allow all origins for endpoints that use credentials.
- Set security headers where relevant, following the project's approach.
- Make the safe option the default in your own code too: features off until enabled, access denied until granted.

### 11. Handle files and outgoing requests safely

- **File paths:** never build file paths from user input without strict validation. Resolve the final path and confirm it stays inside the intended directory.
- **Uploads:** limit size, check the actual content type (not just the extension or the client's claim), generate your own file names, and store uploads outside executable or publicly served locations.
- **Outgoing requests to user-supplied URLs:** allow-list destinations where possible, and block requests to internal addresses, so attackers can't use your server to reach internal systems.

### 12. Limit resource use

- Rate-limit endpoints, especially sign-in, sign-up, password reset, and anything expensive.
- Set timeouts on requests and external calls, and limits on payload sizes, page sizes, and query complexity.
- Avoid operations whose cost an attacker can make grow without bound, such as unbounded loops or regular expressions vulnerable to catastrophic backtracking on user input.

### 13. Log security events

Record security-relevant events so attacks and misuse can be detected: sign-ins and failed sign-ins, permission denials, password and permission changes, and administrative actions. Include who, what, when, and from where, and never the secrets themselves.

### 14. Build layers of defence

Assume any single protection can fail. Validate input *and* use parameterised queries *and* restrict database permissions. Check permissions in the service *and* scope queries to the user. One mistake should never be enough for a breach.

### 15. Never weaken security to make something work

When something fails because of a security control, fix the cause, not the control.

- Never disable certificate verification, authentication, authorisation, CSRF protection, input validation, or security headers to get past an error.
- Never set permissions to "allow everything" (CORS `*` with credentials, file permissions `777`, database superuser) as a workaround.
- Never commit a real secret, even temporarily or "just for testing".
- If a security control seems to be blocking legitimate work, stop and explain the situation to the person you are working with, and let them decide.

## Before finishing

For any change that touches input, identity, data, or network exposure:

- [ ] All external input is validated on the server, with size limits.
- [ ] No query, command, or HTML is built by combining strings with input.
- [ ] Every new operation checks that this user may act on this specific object.
- [ ] No secrets in code, logs, tests, or error messages.
- [ ] Responses return only the fields needed.
- [ ] Any new dependency is necessary, legitimate, maintained, and pinned.
- [ ] Nothing was disabled or loosened to make the change work.
- [ ] Security-relevant tests exist: unauthenticated access, another user's data, invalid input.

## Example

**Poor:**

```
handleGetInvoice(request):
    id = request.query.id
    invoice = database.query("SELECT * FROM invoices WHERE id = " + id)
    log("Fetched invoice for token " + request.headers.authorization)
    return invoice
```

Problems: SQL injection through `id`. No check that the requester owns the invoice, so any user can read any invoice by changing the ID. A bearer token is written to the logs. Every column is returned, including internal fields.

**Better:**

```
handleGetInvoice(request):
    user = request.authenticatedUser                    # from verified session, never from the request body
    invoiceId = InvoiceId.parse(request.params.id)      # rejects anything that isn't a valid ID

    invoice = invoices.findForOwner(invoiceId, ownerId=user.id)   # parameterised, scoped to the owner
    if invoice is absent:
        raise NotFound                                   # same response whether it doesn't exist or isn't theirs

    log.info("invoice.viewed", invoiceId=invoice.id, userId=user.id)
    return toInvoiceResponse(invoice)                   # only the fields the client needs
```

## Common mistakes

- Concatenating input into SQL, shell commands, or HTML.
- Checking that a user is signed in but not that they own the record.
- Trusting `userId`, `role`, or `isAdmin` values sent by the client.
- Relying on the interface to hide actions users aren't allowed to perform.
- Storing passwords with fast hashes, or reversibly.
- Hard-coding secrets, or committing `.env` files.
- Logging tokens, passwords, or full request bodies.
- Returning whole database records, including internal or sensitive fields.
- Adding unfamiliar or brand-new packages without checking them.
- Disabling TLS verification or loosening CORS to fix an error.
- Writing a custom authentication or encryption scheme.
- Missing rate limits on sign-in and password reset.

## Reference files

| File | Read when |
|---|---|
| `references/input-and-injection.md` | Handling input, building queries or commands, rendering output, uploads, file paths, or outgoing requests |
| `references/authentication-and-sessions.md` | Working on sign-in, sign-up, passwords, tokens, sessions, or account recovery |
| `references/authorization.md` | Adding endpoints or operations, or checking who may access what |
| `references/secrets-and-data.md` | Handling secrets, keys, personal or payment data, or cryptography |
| `references/dependencies-and-configuration.md` | Adding packages, configuring environments, CORS, headers, or production settings |

## Sources

OWASP Top 10; OWASP API Security Top 10; OWASP Application Security Verification Standard (ASVS); OWASP Cheat Sheet Series (Input Validation, SQL Injection Prevention, Cross-Site Scripting Prevention, Authentication, Authorization, Password Storage, Session Management, Secrets Management, File Upload, Server-Side Request Forgery Prevention, Logging); NIST SP 800-63B, *Digital Identity Guidelines*; Jerome Saltzer and Michael Schroeder, "The Protection of Information in Computer Systems" (least privilege, fail-safe defaults); CWE Top 25 Most Dangerous Software Weaknesses.