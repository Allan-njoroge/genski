# Dependencies and Configuration

Software is only as secure as the code it depends on and the way it's configured. This file covers choosing and maintaining dependencies, and configuring applications safely.

## Contents

1. Choosing dependencies
2. Maintaining dependencies
3. Supply chain safety
4. Secure configuration
5. Cross-origin access (CORS)
6. Security headers
7. Rate limiting and resource limits
8. Security logging

## 1. Choosing dependencies

Before adding a package, check:

- **Is it needed?** Prefer the standard library or a package the project already uses. A few lines of your own code can be safer than a dependency for a trivial task.
- **Is it real?** Confirm the exact name on the official registry. Attackers publish packages with names that imitate popular ones (typosquatting) or that match names AI tools commonly invent.
- **Is it maintained and trusted?** Look for recent maintenance, broad usage, a public repository, and a responsive maintainer.
- **What does it pull in?** Large dependency trees increase risk.
- **Is the version mature?** Avoid versions published in the last couple of weeks, which is when malicious releases are most often caught and pulled.

Ask the person you're working with before adding a dependency the project doesn't already use, unless the project's guidelines say otherwise.

## 2. Maintaining dependencies

- **Pin versions** and commit the lock file, so builds are reproducible and no unreviewed version slips in.
- **Run the project's vulnerability audit** (the package manager's audit command, or a scanning tool) and address serious findings.
- **Update regularly,** in small steps, with tests, so updates don't pile up into a risky upgrade.
- **Remove unused dependencies.**

## 3. Supply chain safety

- Install from official registries over HTTPS.
- Be cautious with packages that run install scripts.
- Don't load scripts from third-party CDNs in production without pinning exact versions, and use integrity checks where supported.
- Keep build and deployment credentials scoped and protected. Build systems are a common target.

## 4. Secure configuration

- **Production defaults are secure:** debug mode off, detailed error pages off, development tools and test endpoints absent.
- **Configuration comes from the environment,** not from code, and is validated at startup.
- **Expose only what's needed:** close unused ports, disable unused features, and don't expose administrative interfaces publicly.
- **Run with least privilege:** don't run processes as root or administrator, and give each service its own credentials.
- **Keep environments separate:** development and test environments never use production data or secrets unless the project's process explicitly requires it.

## 5. Cross-origin access (CORS)

CORS controls which websites' scripts may call your API from a browser.

- Allow only the specific origins that need access.
- Never combine "allow any origin" with credentials (cookies or authorisation headers).
- Don't reflect the request's `Origin` header back as allowed without checking it against an allow-list.
- Allow only the methods and headers needed.
- Remember that CORS protects browsers' users. It is not authentication, and doesn't stop non-browser clients. Authorisation checks are still required.

## 6. Security headers

For web applications and APIs serving browsers, set headers that enable browser protections. Common ones:

| Header | Purpose |
|---|---|
| `Strict-Transport-Security` | Forces HTTPS for future visits |
| `Content-Security-Policy` | Restricts where scripts and other resources can load from, limiting XSS |
| `X-Content-Type-Options: nosniff` | Stops browsers from guessing content types |
| `Referrer-Policy` | Limits URL information sent to other sites |
| `frame-ancestors` in CSP (or `X-Frame-Options`) | Prevents the site being embedded to trick users (clickjacking) |

Use the framework's or project's middleware for headers rather than setting them by hand in each response.

## 7. Rate limiting and resource limits

- Rate-limit by user, by API key, and by source address, as appropriate, with stricter limits on sign-in, sign-up, password reset, one-time code verification, and expensive operations.
- Return 429 Too Many Requests when limits are reached.
- Set maximum sizes for request bodies, uploads, and page sizes, and timeouts on requests and external calls.
- Paginate every list. Never return unbounded collections.

## 8. Security logging

Log enough to detect and investigate attacks:

- Successful and failed sign-ins, sign-outs, and session revocations.
- Password, email, and multi-factor changes.
- Permission denials and failed authorisation checks.
- Changes to roles and permissions, and administrative actions.
- Rate limit triggers and rejected input that suggests probing.

Each entry should record who, what, when, from where (such as source address), and the outcome, without secrets or unnecessary personal data. Protect logs from tampering and from unauthorised reading.