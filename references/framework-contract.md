# Contract / invariant reviewer

Source: Meyer's Design by Contract (Eiffel, 1992); Hoare logic (preconditions, postconditions, invariants); Liskov Substitution Principle. The core insight: code makes implicit promises that callers depend on. Bugs arise when code VIOLATES its own contract, or when a change SILENTLY ALTERS a contract that callers still rely on.

## The model

Every function/method/module has a contract, whether documented or not:

- **Preconditions**: what must be true BEFORE this code runs (caller's responsibility).
- **Postconditions**: what this code guarantees AFTER it runs (callee's promise).
- **Invariants**: what must ALWAYS be true about the object/module state, before and after every public operation.

A bug is a contract violation: a postcondition that is not actually maintained, a precondition that callers can violate, or an invariant that a method temporarily breaks and fails to restore.

## How to extract the contract

The contract is NOT what the doc comment says. Extract it from:

1. **The type signature**: return type (nullable? error-union? optional?), parameter types (their ranges, what they accept).
2. **The tests**: what inputs the tests use and what they assert tells you what the code promises.
3. **The callers**: what callers assume (do they check for null? do they expect sorted output? do they rely on ordering?).
4. **The implementation**: what the code actually ensures vs what it could theoretically violate.
5. **The name**: a function named `getOrCreate` promises it always returns something. `ensureValid` promises validity. `tryConnect` is allowed to fail.

## What to check in the diff

### Postcondition violations (the code lies about what it returns)

1. **Can this function return something its callers do not expect?** Null when callers assume non-null. An error variant callers do not handle. A different shape than before.
2. **Does a new code path skip the postcondition?** Early return that bypasses the "setup" the rest of the function does. Error path that returns partial/default state. Timeout path that returns stale data.
3. **Does the function's name still match its behavior?** If `validate()` now also modifies, or `get()` now has side effects, or `isReady()` can block.
4. **Does a conditional change silently narrow the postcondition?** "Used to always return a list, now returns empty on a new condition that callers don't check for."

### Precondition weakening (callers can now pass bad input)

5. **Did the function become more permissive without callers knowing?** Accepting null where it used to reject, accepting a wider range that downstream cannot handle.
6. **Is there a new caller that violates the existing precondition?** The function assumes sorted input; the new caller does not sort.

### Invariant violations (object/module state becomes inconsistent)

7. **Does a new method leave the object in a state that other methods assume it cannot be in?** (e.g., a list that is supposed to be sorted after every mutation, but a new bulk-insert skips sorting.)
8. **Does an error path leave partial state?** (Transaction half-committed, struct half-initialized, cache half-invalidated.)
9. **Does concurrent access break an invariant that sequential access maintains?** (Two threads both read-modify-write a "counter" that must never go below zero.)

### Contract drift (the change alters a contract callers depend on)

10. **What did callers rely on about the OLD behavior that the NEW behavior no longer provides?** This is the #1 class of bugs this lens catches. The change works in isolation; it breaks callers. Look at every caller and ask: "what did they assume that is no longer true?"
11. **Is a return type widened/narrowed?** Adding a new possible return value (e.g., a new enum variant, a new error type) that existing match/switch statements do not handle.
12. **Is ordering, uniqueness, or completeness no longer guaranteed?** (Used to return results sorted; now returns them in insertion order. Used to be deduplicated; now may contain dupes.)

## How a finding should read

State the contract, state how the diff breaks it, state who depends on it. "The function contract (inferred from 3 callers and the return type) guarantees non-null return. The new early-return at line 87 returns null on the `connectionTimeout` path. Callers `ProcessBatch` (line 22) and `RetryLoop` (line 55) both dereference the result without null checks."
