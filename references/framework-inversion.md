# Inversion reviewer

Source: Carl Jacobi ("invert, always invert"); Charlie Munger ("all I want to know is where I'm going to die, so I'll never go there"). Forward reasoning confirms the author's intent and inherits their blind spots. Inverse reasoning hunts the conditions of failure, which is exactly what forward reasoning skips.

## The stance

Treat every changed line as a CLAIM of correctness and try to break it. Two paired questions per finding:

1. What input / state / ordering / environment would make this code wrong?
2. For this to be correct, what must ALWAYS hold, and is that actually enforced, or just assumed?

The second is load-bearing: it turns a vague worry into a checkable invariant, then asks whether the diff enforces it or merely trusts it. "Usually true" is the tell of a latent bug.

## Prompts to run against the diff

1. **What input breaks this?** Empty, null, zero, negative, max-int, duplicate, unsorted, oversized, malformed encoding.
2. **What must always be true here, and is it enforced or assumed?** Name the precondition, then grep the diff for the check. If it is a comment or a hope, flag it.
3. **How would I make this return wrong-but-plausible?** Silent truncation, wrong default on miss, swallowed error, fallback that masks failure, units/sign confusion. Wrong-but-plausible is worse than a crash; it passes tests.
4. **What does it assume about caller/env that it does not check?** Auth already verified upstream, config present, env var set, clock monotonic, locale/timezone, flag on, dependency version.
5. **Which reachable branch is unhandled?** The else that does nothing, the default case, the early return that skips cleanup, the exception that escapes. Ask: unreachable, or just untested?
6. **If two run at once, what corrupts?** Check-then-act races, shared mutable state, missing idempotency on retry, ordering between async calls, partial writes.
7. **What makes cleanup/rollback not run?** Leaked handles/locks/conns on the error path, txn that commits partial work, resources freed only on success.
8. **What would have to be true for this to NOT fail, and is it guaranteed or merely usual?** Restate the author's implicit safety argument out loud, then attack each clause.
9. **Where does it trust data it did not produce?** External responses, user input, deserialized payloads, DB rows written by older code. Hostile or stale?
10. **What did this change silently make true elsewhere?** New nullable column, changed default, widened type, removed validation, altered call order. Who downstream relied on the old guarantee?

## How a finding should read

State the inverse, not the intent. Not "this looks fine", but "this is correct only if `list` is non-empty; nothing in the diff enforces that, and caller Foo can pass empty, so `[0]` throws." Always name: the assumed invariant, the input/state/order that violates it, and whether the diff enforces it or just relies on it.
