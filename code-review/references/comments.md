# Writing Review Comments

The value of a review depends on how well its comments are understood and acted on. A correct point made badly may be ignored, misunderstood, or resented.

## Contents

1. Anatomy of a good comment
2. Examples
3. Labels
4. Tone
5. Responding to comments

## 1. Anatomy of a good comment

A strong comment has up to four parts:

1. **Label:** how important it is (blocking, suggestion, question, nit, praise).
2. **Observation:** what you see, precisely, at a specific location.
3. **Reason:** why it matters: the consequence, risk, or principle.
4. **Suggestion:** what to do instead, ideally with a short example.

```
**Blocking:** `findInvoice` looks up the invoice by ID only. Any signed-in
user could read another customer's invoice by changing the ID in the URL.
Suggest scoping the query to the current user:
    invoices.findForOwner(invoiceId, ownerId=user.id)
```

## 2. Examples

| Weak | Strong |
|---|---|
| "This is wrong." | "**Blocking:** When `items` is empty, `items[0]` throws. The cart page can call this with an empty cart. Could we return an empty summary early?" |
| "Use a constant." | "**Nit:** `86400` appears three times here. A named constant like `SECONDS_PER_DAY` would make the intent clear." |
| "Why did you do it this way?" | "**Question:** This retries on every error, including validation errors. Is that intended? Retrying invalid input will fail the same way each time." |
| "This is too complicated." | "**Suggestion:** `OrderProcessorFactory` and `BaseOrderProcessor` each have one implementation. Calling `OrderProcessor` directly would remove two layers without losing anything. Happy to leave it if more processors are planned soon." |
| "Needs tests." | "**Should fix:** There's no test for the case where the payment provider times out, which is the path most likely to cause double charges. A test with the fake gateway returning a timeout would cover it." |
| (no comment on good work) | "**Praise:** The `InMemoryOrderRepository` makes these tests much clearer than the mocks we had before." |

## 3. Labels

Use the labels defined in `SKILL.md` (blocking, should fix, suggestion, nit, question, praise). If the project uses a different convention, such as Conventional Comments (`issue:`, `suggestion:`, `nitpick:`, `question:`, `praise:`, with optional `(blocking)` or `(non-blocking)`), follow it.

What matters is that every comment's importance is unambiguous.

## 4. Tone

- **Talk about the code, not the person.** "This function does three things" rather than "You wrote a function that does three things."
- **Prefer questions and suggestions to commands** for non-blocking points: "What do you think about…?", "Could we…?"
- **Be direct about blocking issues.** Politeness doesn't require vagueness. "This will lose data if the second write fails" is respectful and clear.
- **Assume good intent and competence.** The author may have context you don't. Ask before assuming a mistake.
- **Avoid loaded words** such as "obviously", "just", "simply", "lazy", or "wrong" on its own. They add judgement without information.
- **Keep it proportionate.** Twenty nits on a small change is discouraging. Pick the ones that matter, or group them.

## 5. Responding to comments

When you are the author:

- **Reply to every comment,** even briefly: "Done", "Fixed in the latest commit", or a reason for not changing it.
- **Explain disagreement** with reasons and trade-offs, not just "I prefer it this way". If the discussion stalls, suggest a quick conversation or ask a third person.
- **Don't take it personally.** Comments are about the code.
- **Say what changed** when you push fixes, so the reviewer knows where to look.
- **Mark threads resolved** only when the point has been addressed or agreed, following the project's convention for who resolves.