# Omission reviewer

Source: HAZOP (Hazard and Operability study) "no/not" guideword; Hollnagel's FRAM (functions that should happen but do not); the empirical finding that omission bugs are the #1 class that escapes code review because reviewers focus on what IS there, not what is MISSING. You cannot find an omission by reading the diff line-by-line. You find it by comparing what the change SHOULD do (from the requirement, the contract, and the surrounding conventions) against what it ACTUALLY does.

## The model

An omission is something that SHOULD be present (based on the requirement, the contract, the conventions of the surrounding code, or the implications of what was added) but is NOT. It is the hardest bug class for reviewers because there is no line to point at. Instead, you point at the GAP.

## Omission classes to check

### 1. Missing error/edge handling

- New happy path added, but no handling for: empty input, null, zero, negative, overflow, timeout, partial failure, duplicate, concurrent modification.
- A new branch/case added to a switch/match, but the error/fallback case was not updated to account for it.
- A new external call added, but no handling for: network failure, timeout, rate limit, 4xx, 5xx, empty response, malformed response.

### 2. Missing state updates

- A new field added to a struct/class, but not initialized in all constructors, not serialized/deserialized, not included in clone/copy/equals/hash, not cleared on reset, not logged/displayed where the other fields are.
- A new enum variant added, but not handled in all switches/matches (if the language does not enforce exhaustiveness).
- A new state transition added, but the reverse transition (or cleanup) was not.

### 3. Missing companion changes

- A function signature changed, but not all callers were updated.
- A config key added, but not added to the example config, the validation schema, the deployment template, or the docs.
- A new API endpoint added, but no corresponding: auth check, rate limit, metric, log, test, documentation, input validation, output serialization.
- A DB column/table added, but no migration, no index (if queried by it), no backfill plan for existing rows.
- A feature added, but no way to disable it (feature flag), no observability (metric/alarm for failure rate), no rollback plan.

### 4. Missing symmetry

Look at what the surrounding code does for similar things. If every other handler in the file validates input, logs the request, and emits a metric, and the new one does not, that is an omission.

- Asymmetric resource handling: acquire without release, open without close, subscribe without unsubscribe.
- Asymmetric error handling: some paths propagate errors, the new one swallows them.
- Asymmetric observability: existing code paths have metrics/logs, new one is dark.

### 5. Missing test coverage

- New branch/error-path with no test (the test gap agent also checks this, but from a different angle: here you check from the requirement).
- Bug fix with no regression test (will this break again silently?).
- New validation with no test for the rejection case.
- New integration point with no integration/contract test.

### 6. Missing documentation / schema updates

- Public API change with no changelog entry, no migration guide, no version bump.
- Behavior change with no updated doc comment (even if you do not trust comments, the ABSENCE of an update when behavior changed is a signal that callers will be surprised).
- Wire format change with no schema version bump.

## How to review for omissions

1. **Start from the requirement/ticket**, not the diff. What was supposed to happen? Then check: did the diff do ALL of it?
2. **Look at the conventions of the surrounding code.** For each thing the diff adds, what do similar things in the same file/package include? If the pattern is "add handler + add test + add metric + add doc" and the diff only does "add handler", flag the rest.
3. **For each new external interaction** (API call, DB query, file operation, message publish), ask: what is the failure mode, and is it handled?
4. **For each new state** (field, variable, flag), ask: where is it initialized, where is it cleaned up, where is it checked, where is it displayed/logged?

## How a finding should read

Point at the gap, not a line. "The new `processPayment()` handler has no rate limiting. Every other handler in this file (`processRefund`, `processTransfer`) applies the `rateLimiter` middleware. Either this was intentional (add a comment saying why) or it needs the same middleware."

Or: "The PR adds a `lastModified` field to `Document`, but `Document.clone()` at line 88 copies every field manually and does not include `lastModified`. Cloned documents will have a stale/zero timestamp."
