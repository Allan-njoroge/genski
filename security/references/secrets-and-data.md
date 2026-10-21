# Secrets, Cryptography, and Sensitive Data

This file covers handling secrets, using cryptography correctly, and protecting personal and sensitive data.

## Contents

1. Handling secrets
2. When a secret leaks
3. Cryptography
4. Random values
5. Sensitive and personal data
6. Data in logs, errors, and analytics

## 1. Handling secrets

Secrets include passwords, API keys, tokens, private keys, signing keys, database connection strings, and webhook signing secrets.

- **Never in code or version control:** not in source, committed configuration, tests, fixtures, documentation, examples, or comments.
- **Load at runtime** from environment variables or a secrets manager, as the project does.
- **Keep local secret files ignored:** add files like `.env` to the version-control ignore file, and commit a template (`.env.example`) with placeholder values only.
- **Separate per environment:** development, staging, and production use different secrets. Production secrets never appear on developer machines unless the project's process requires it.
- **Fail at startup** if a required secret is missing, rather than continuing with a default.
- **Never use real secrets in tests.** Use obviously fake values.
- **Don't print secrets,** including in debug output, error messages, or command output.

## 2. When a secret leaks

If a secret has been committed, logged, pasted, or otherwise exposed:

1. Treat it as compromised, even if the exposure seems brief or private.
2. Tell the person you're working with that it must be **revoked and replaced**. Removing it from the latest code is not enough, because it remains in version-control history, logs, and caches.
3. Check for any use of the leaked secret, if the provider offers access logs.
4. Removing it from history is a secondary clean-up step, not the fix.

Secret-scanning tools in version control and continuous integration help catch leaks before they're pushed.

## 3. Cryptography

- **Don't design your own.** Use well-maintained libraries and their high-level interfaces.
- **Passwords:** use password hashing algorithms (Argon2id, scrypt, bcrypt). See `authentication-and-sessions.md`.
- **Encryption:** use authenticated encryption, such as AES-GCM or ChaCha20-Poly1305, through a high-level library. Never use outdated algorithms (DES, RC4) or modes without integrity protection (such as ECB).
- **Hashing for integrity or identifiers:** use SHA-256 or stronger. MD5 and SHA-1 are broken for security purposes.
- **Signing data** (cookies, webhooks, links): use HMAC with a secret key, or digital signatures. Verify before trusting the data.
- **Compare secrets in constant time,** using the library's comparison function, to avoid leaking information through timing.
- **Keys:** store them like other secrets, and plan for rotation.
- **Transport:** use TLS for all network traffic, including internal traffic where possible. Never disable certificate verification, even in development code that might be copied.

## 4. Random values

- Use the language's cryptographically secure random generator for anything security-related: tokens, session IDs, reset links, API keys, one-time codes, and unguessable identifiers.
- Ordinary random functions (often named `random` or `Math.random`) are predictable and must not be used for these purposes.
- Make tokens long enough to be unguessable; 128 bits of randomness or more is a common minimum.

## 5. Sensitive and personal data

- **Minimise:** collect only the data you need, and keep it only as long as needed.
- **Classify:** know which fields are sensitive (personal data, financial data, health data, credentials) and treat them accordingly.
- **Restrict access:** only the parts of the system and the people who need sensitive data should be able to read it.
- **Encrypt at rest** where the data's sensitivity or the project's requirements call for it, in addition to encryption in transit.
- **Payment card data:** avoid handling it directly. Use the payment provider's hosted fields or tokens, so card numbers never reach your servers.
- **Follow applicable laws and policies,** such as data protection regulations, as specified by the project. Ask the person you're working with if requirements are unclear.

## 6. Data in logs, errors, and analytics

- Never log passwords, tokens, keys, session IDs, full card numbers, security codes, or one-time codes.
- Avoid logging full request or response bodies, which often contain these. Log selected fields instead.
- Mask or omit personal data, and log identifiers rather than names, addresses, phone numbers, or message contents.
- Make sure error messages and API responses don't include sensitive data or internal details.
- Apply the same care to analytics events, error-tracking tools, and any third-party service that receives data.