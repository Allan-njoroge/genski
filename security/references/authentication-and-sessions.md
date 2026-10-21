# Authentication and Sessions

Authentication establishes who someone is. Sessions and tokens keep them signed in. Both are easy to get subtly wrong, which is why the first rule is to rely on proven implementations.

## Contents

1. Use established solutions
2. Passwords
3. Sign-in protection
4. Multi-factor authentication
5. Account recovery
6. Sessions
7. Tokens (including JWTs)
8. API keys and service credentials

## 1. Use established solutions

- Use the project's authentication library, framework features, or identity provider. Don't write your own sign-in, session, or token scheme.
- When adding authentication to a project that has none, recommend a well-maintained option and let the person you're working with choose.
- Keep authentication logic in one place, applied consistently through middleware or the framework's mechanism.

## 2. Passwords

**Storage:**

- Hash with an algorithm designed for passwords: **Argon2id** (preferred), **scrypt**, or **bcrypt**. Use the settings recommended by the current OWASP Password Storage Cheat Sheet, or the library's secure defaults.
- These algorithms salt automatically. Never use fast general-purpose hashes (MD5, SHA-1, SHA-256) alone, and never encrypt passwords reversibly.
- Verify with the library's comparison function, which is designed to avoid timing leaks.

**Rules for users** (following NIST SP 800-63B):

- Require a reasonable minimum length, and allow long passwords (at least 64 characters) and all characters, including spaces.
- Check new passwords against lists of known breached or very common passwords.
- Don't impose composition rules (such as "must include a symbol") or force periodic password changes without reason. They lead to weaker, predictable passwords.
- Allow paste into password fields, so password managers work.

## 3. Sign-in protection

- **Rate-limit** sign-in attempts per account and per source, with increasing delays or temporary lockouts. Be careful that lockouts can't be used to lock out legitimate users at will.
- **Give the same response** whether the account doesn't exist or the password is wrong: "Incorrect email or password."
- Keep response timing similar in both cases where practical.
- **Log** failed and successful sign-ins, without the password.
- Consider CAPTCHA or similar challenges after repeated failures, if the project uses them.

## 4. Multi-factor authentication

- Support or require multi-factor authentication for sensitive accounts and actions, especially administrators.
- Prefer authenticator apps, security keys, and passkeys. SMS codes are better than nothing, but weaker.
- Require re-authentication for sensitive actions such as changing email, password, payment details, or security settings.

## 5. Account recovery

Recovery flows are a frequent weak point.

- Respond the same way whether or not the account exists: "If an account exists for this email, we've sent a link."
- Use reset tokens that are long, random (from a secure generator), **single-use**, and **short-lived**.
- Store only a hash of the reset token, as with passwords.
- After a successful reset, invalidate existing sessions and notify the user.
- Never send passwords by email.

## 6. Sessions

- Generate session identifiers with a secure random generator; the framework normally does this.
- **Regenerate the session ID on sign-in** and on privilege changes, to prevent session fixation.
- For browser sessions, use cookies marked:
  - `HttpOnly` (scripts can't read them),
  - `Secure` (sent only over HTTPS),
  - `SameSite=Lax` or `Strict` (limits cross-site sending).
- Expire sessions after inactivity and after a maximum lifetime.
- Invalidate sessions on sign-out, on password change, and when the account is disabled.
- Protect state-changing requests from cross-site request forgery (CSRF) using the framework's protection or `SameSite` cookies, as appropriate.

## 7. Tokens (including JWTs)

- Keep access tokens short-lived. Use refresh tokens to obtain new ones, and make refresh tokens revocable, ideally rotating them on each use.
- **Always verify the signature** and check expiry, issuer, and audience, using a maintained library.
- Fix the accepted algorithm on the server. Never accept a token that declares "no algorithm", and never let the token's header decide how it's verified.
- Don't put sensitive data in a JWT's payload. It's encoded, not encrypted; anyone holding the token can read it.
- In browsers, avoid storing tokens where any script can read them (such as `localStorage`) when an `HttpOnly` cookie is an option.
- Have a way to revoke access, such as short lifetimes plus a revocation list or token versioning, for when an account is compromised.

## 8. API keys and service credentials

- Generate keys with a secure random generator and show them to the user once.
- Store only a hash of each key, so a database leak doesn't expose usable keys.
- Scope keys to specific permissions, and let users name, list, rotate, and revoke them.
- Use separate credentials for each service and environment, so one leak doesn't compromise everything.