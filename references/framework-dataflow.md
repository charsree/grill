# Data-flow / taint tracking reviewer

Source: adapted from taint analysis in static analysis (FlowDroid, Infer, CodeQL), formalized as source-sink-sanitizer triples. The core insight: bugs hide in the FLOW of data through code, not in any single line. A value that is safe at its origin becomes dangerous when it reaches a context that trusts it.

## The model

Every piece of data has a TRUST LEVEL based on where it came from (its source). As it flows through the code, operations either preserve, elevate, or should check that trust level. A bug occurs when untrusted data reaches a trusting context (a sink) without passing through adequate validation (a sanitizer).

## Sources (untrusted origins, track these)

| Source class | Examples |
|---|---|
| User input | HTTP params, form fields, CLI args, file uploads, request headers, cookies, path segments |
| External system responses | API responses, DB query results (if the DB is writable by others), message queue payloads, webhook bodies |
| Deserialized data | JSON.parse, protobuf decode, pickle, YAML, any wire-to-object conversion |
| Environment / config | env vars, config files (especially if modifiable at runtime), feature flags |
| Stored data under external control | DB rows written by users, S3 objects, cache entries that could be poisoned |

## Sinks (trusting contexts, dangerous to reach untrusted)

| Sink class | The danger |
|---|---|
| SQL / query construction | Injection |
| Shell / command execution | Command injection |
| HTML / template rendering | XSS |
| File path construction | Path traversal |
| Redirect URL construction | Open redirect |
| Deserialization target type | Type confusion, RCE |
| Logging with PII/secrets | Data exposure |
| Comparison for auth decisions | Auth bypass |
| Numeric use in allocation/index | Integer overflow, OOB |
| Use as a cache key | Cache poisoning |
| Use in a regex | ReDoS |

## What to check in the diff

For each piece of data that enters modified code:

1. **Trace it forward.** Where does it go? Does it reach any sink? Draw the path: source -> transforms -> sink.
2. **Check the sanitizers on that path.** Is there validation/encoding/escaping between the source and the sink? Is the sanitizer correct for that specific sink type (e.g., HTML encoding does not help SQL injection)?
3. **Check for sanitizer bypass.** Can the data reach the sink on ANY path that skips the sanitizer? (Early returns, error paths, fallback branches, conditional logic that routes around it.)
4. **Check for double-use.** Is the same input used in two sinks that need different sanitization? (Escaped for HTML but also used in a SQL query.)
5. **Check for implicit trust elevation.** Does the code store untrusted data and later read it back as trusted? (Write user input to DB, read it back later without re-validation.)
6. **Check for trust boundary confusion.** Is data from one trust domain used in another without re-validation? (Internal service A passes data to service B, which trusts it because it came from A, but the data originally came from a user through A.)

## Implicit flows (subtle)

Beyond direct data flow, check:
- **Control flow leaks:** an `if` on a secret value followed by observable behavior in each branch (timing, response shape, error message).
- **Length/existence leaks:** checking `if (record exists)` and returning different errors exposes whether a record exists.
- **Error message content:** exceptions or logs that include untrusted values or internal state.

## How a finding should read

Name: the source, the sink, the missing/broken sanitizer, and the path. "User-supplied `projectId` from the URL path flows into `query.filter()` at line 42 without validation against the user's authorized projects. The auth check at line 30 validates the session but does not scope the query."
