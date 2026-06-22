# STAMP / STPA control-hazard reviewer

Source: Leveson, "Engineering a Safer World" (MIT, 2011); STPA Handbook (Leveson and Thomas, 2018). The four unsafe-control-action classes are proven complete in the handbook.

## The model

Treat the changed function/handler/service as a **controller**. The calls it makes are **control actions**. The thing it acts on (resource, peer, state, store) is the **controlled process**. Return values, errors, events, and reads are **feedback** that update the controller's **process model** (its local state and assumed invariants). A bug is inadequate control: the controller acts on a process model that does not match reality.

## The 4 unsafe-control-action (UCA) classes, mapped to code

| Class | Code smell to hunt |
|---|---|
| 1. Required action NOT provided | Missing state transition, missing guard/validation, an early return/break/throw that skips a required commit/cleanup/release/ack/notify. Check EVERY branch, especially error paths. |
| 2. Action provided when unsafe | A write/send/delete/mutation on a path or in a state where it must not happen; missing precondition; wrong parameter (off-by-one, wrong id, wrong sign/units); excessive/repeated (retry storm, double-submit, no idempotency). |
| 3. Wrong time / wrong order | use-before-init, read-before-write, commit-before-flush, callback fired before subscription ready; check-then-act / TOCTOU races; awaits in wrong order. |
| 4. Held too long / stopped too soon | Lock/connection/txn/handle held across a slow call or never released; OR a resource freed/closed/cancelled while still in use; timer/subscription/loop ended too early. |

## Feedback / process-model flaws (the bug-rich part for software)

Flag when the controller:
- receives **incorrect** feedback (parses/interprets a result wrong)
- **ignores** correct feedback (unchecked return value, empty `catch`, `let _ =`)
- assumes a **prior action succeeded** without checking (fire-and-forget write, unawaited future, unverified RPC)
- relies on feedback that is **missing, delayed, or stale** (no timeout, reads a cache that may be old)

## Review questions to ask the diff

1. (Class 1) On every path including each error/early-exit, does the state change / cleanup / ack that should happen actually happen? Which branch skips it?
2. (Class 2) Is there a state where this new write/send/delete is not allowed, and is that guarded? What is the precondition, is it checked here?
3. (Class 2) Are args right in value, sign, direction, units? Could a repeated/excessive call harm (idempotency, debounce, backoff)?
4. (Class 3) Does this assume an ordering (init-before-use, lock-before-access, subscribe-before-publish) it does not enforce? Any check-then-act gap?
5. (Class 4) For each resource acquired (lock, conn, txn, handle, task), is it released exactly once on every path, not too late, not too early?
6. (Feedback) Is any return/error/status ignored, swallowed, or logged-and-continued?
7. (Feedback) Does it assume its previous action succeeded without checking? What if feedback is delayed/stale/never?
8. (Interactions) Can another thread/service/caller act on the same process concurrently, and does this code detect or resolve the conflict, or assume it is the only actor?

## Stance

- **Worst-case, not most-likely.** Do not dismiss a finding because "a safeguard exists" or "that branch rarely runs." Flag it; the refutation pass decides if a real guard kills it.
- **Always name the context.** A finding is "unsafe WHEN <state>". That condition is also the repro and the test case. A UCA with no triggering context is not a finding.
