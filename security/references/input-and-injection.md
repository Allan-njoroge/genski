# Input and Injection

Injection vulnerabilities happen when untrusted data is treated as part of a command, query, or document. This file covers validating input and preventing each common kind of injection.

## Contents

1. Validating input
2. SQL and database queries
3. NoSQL and other query languages
4. Shell commands
5. Cross-site scripting (HTML and JavaScript output)
6. Templates and dynamic code
7. Deserialisation
8. File paths
9. File uploads
10. Outgoing requests (server-side request forgery)
11. Regular expressions and resource exhaustion

## 1. Validating input

- **Validate on the server, at the boundary,** using the project's schema or validation library.
- **Allow-list, don't block-list.** Define exactly what's valid: types, formats, lengths, ranges, allowed values. Rejecting "known bad" patterns always misses something.
- **Validate structure and meaning.** An ID must be the right format *and* refer to something the user may access. A date must parse *and* fall within a sensible range.
- **Reject unknown fields** where the framework allows, so attackers can't set fields you didn't intend to accept (mass assignment), such as `role` or `isVerified`.
- **Limit size** of every input: body size, string length, array length, nesting depth, number of items per page.
- **Normalise before validating** where relevant (for example, Unicode normalisation, trimming), and validate the normalised value.

Validation reduces risk, but it is not a substitute for the context-specific protections below. Use both.

## 2. SQL and database queries

Always separate query structure from data.

```
Vulnerable:
    db.query("SELECT * FROM users WHERE email = '" + email + "'")
    db.query(`SELECT * FROM orders WHERE id = ${id}`)

Safe:
    db.query("SELECT id, email FROM users WHERE email = $1", [email])
    orders.where({ id: id })          # query builder or ORM binding
```

- Use parameterised queries or the ORM or query builder's binding for every value.
- ORMs are safe by default, but most offer "raw" query functions. Use parameters with them too.
- Identifiers such as column names and sort directions can't be parameterised. If a user chooses them (for example, a sort field), map their choice through an allow-list: `{"date": "created_at", "amount": "total"}`.
- Use a database account with only the permissions the application needs.

## 3. NoSQL and other query languages

The same principle applies to document databases, search engines, LDAP, XPath, and GraphQL.

- Never build queries by concatenating strings.
- In document databases, make sure input values can't be objects containing query operators. If a field should be a string, validate that it is a string, so `{"$ne": null}` can't be sent where an email was expected.
- For GraphQL, limit query depth, complexity, and result sizes, and apply authorisation in each resolver.

## 4. Shell commands

- **Prefer libraries over shell commands.** Most tasks (file operations, image processing, compression) have libraries that avoid the shell entirely.
- If a command must run, call the program directly with arguments as a list, using a function that does not invoke a shell interpreter.
- Never pass user input to a function that runs a command string through a shell.
- Allow-list any user-influenced values, and make sure arguments can't be interpreted as options (for example, a value starting with `-`).

## 5. Cross-site scripting (HTML and JavaScript output)

Cross-site scripting (XSS) lets attackers run their scripts in other users' browsers.

- Use a UI framework or template engine that escapes output by default, and keep that default.
- Never insert untrusted content using raw HTML features (such as `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, or "unescaped" template tags).
- When rich text from users is required, sanitise it with a well-maintained HTML sanitiser configured with an allow-list of tags and attributes.
- Escape for the right context: HTML body, HTML attributes, JavaScript, CSS, and URLs each need different encoding. Frameworks handle this when used normally.
- Validate URLs that users supply for links or images: allow only `https:` (and `http:` if needed), never `javascript:` or `data:` from untrusted sources.
- A Content Security Policy header adds a strong extra layer of protection.

## 6. Templates and dynamic code

- Never pass user input to `eval`, `Function` constructors, `exec`, or similar dynamic execution features.
- Never let users provide template source code. Users supply *data* to templates, never templates themselves.
- Never use user input to choose which module, class, or file to load, except through a fixed allow-list.

## 7. Deserialisation

Some serialisation formats can create arbitrary objects or run code when decoded (for example, native object serialisation in several languages, and unsafe YAML loading).

- Accept untrusted data only in plain data formats such as JSON.
- Use "safe" loading modes for formats such as YAML.
- Validate the decoded data against a schema before use.
- Never deserialise data that has been stored where users could modify it (cookies, hidden fields) unless it is signed and the signature is verified.

## 8. File paths

Path traversal lets attackers read or write files outside the intended directory with input such as `../../etc/passwd`.

- Don't build paths from user input if you can avoid it. Use generated identifiers and look up the real path.
- If you must, allow-list the characters, join the input to a fixed base directory, resolve the final absolute path, and check it still starts with the base directory.
- Reject absolute paths, `..` segments, and null bytes in user-supplied names.

## 9. File uploads

- Limit file size and the number of files.
- Check the file's actual content type by inspecting it, not by trusting the extension or the client's declared type. Allow only the types the feature needs.
- Generate your own file names. Never use the uploaded name as the storage path.
- Store uploads outside the application's code and outside directories the web server executes or serves directly. Object storage with private access is a common choice.
- Serve user uploads with headers that stop browsers from interpreting them as web pages (for example, a correct `Content-Type`, `Content-Disposition: attachment` where appropriate, and `X-Content-Type-Options: nosniff`).
- Scan for malware where the files will be shared with other users, if the project's requirements call for it.
- Be cautious with archives (size bombs, path traversal inside archives) and with formats that can embed scripts, such as SVG.

## 10. Outgoing requests (server-side request forgery)

When the server fetches a URL a user provides (webhooks, link previews, image imports), an attacker can point it at internal systems, such as cloud metadata services or internal admin panels.

- Allow-list destinations (specific domains) where the feature allows.
- Otherwise, resolve the hostname and block private, loopback, link-local, and other internal addresses, including after redirects.
- Allow only `https:` (and `http:` if necessary). Block other schemes such as `file:`.
- Set short timeouts and response size limits.
- Don't return raw responses or error details from the fetch to the user.

## 11. Regular expressions and resource exhaustion

- Some regular expressions take exponential time on crafted input (catastrophic backtracking). Avoid nested quantifiers such as `(a+)+` on user input, limit input length before matching, or use a regex engine with linear-time guarantees where available.
- Bound every loop, recursion, and batch whose size an attacker can influence.
- Limit decompression, image dimensions, and document parsing to sensible sizes.