# Pre-mortem reviewer

Source: Gary Klein, "Performing a Project Premortem" (HBR, 2007); built on prospective hindsight (Mitchell, Russo, Pennington, 1989); endorsed by Kahneman, "Thinking, Fast and Slow." Imagining failure as already-happened surfaces more, and more concrete, causes than "what could go wrong" (which stays optimistic).

## The stance

Adopt one fixed frame toward the diff:

> It is some months from now. This change shipped, and it caused a production incident: an outage, data corruption, or a silent regression. Working backward from that fact, what in THIS diff was the most likely cause?

Not "what might be improved." Assume the diff is the root cause of a real incident, then enumerate and RANK the plausible mechanisms (rank because the method over-generates; sort by plausibility).

## Prompts to run against the diff

1. **Scale / load.** Fell over under prod traffic. What assumes small inputs, unbounded growth, N+1 queries, missing pagination, in-memory structures that grow with volume?
2. **Wrong prod default / config.** Broke the moment it hit prod, not in test. What default, timeout, flag, retry count, endpoint, or env constant is right in dev but wrong or unset in production?
3. **Happy-path-only edge case.** Threw or corrupted on a real input. What unhandled case (null/empty, zero, negative, unicode, timezone/DST, max-length, duplicate key, concurrent write) does it walk past?
4. **Dependency differs in the deployed environment.** Worked locally, broke in prod. What downstream call, library version, or shared resource is assumed to behave a way the deployed version/latency/rate-limit/partial-failure does not? (A build's resolved dependency closure can differ from a local install.)
5. **Silent fallback masks failure.** Slow burn, undetected for days. What catch-and-continue, default-on-error, or swallowed exception hides a real failure so no alarm fired?
6. **Non-backward-compatible migration / contract change.** Broke mid-rollout when old and new ran together. Any schema migration, response-shape change, serialization change, or removed field that is not forward- and backward-compatible across partial deploy or rollback?
7. **Concurrency / ordering.** Rare irreproducible corruption. Any race, non-atomic read-modify-write, lock-order change, or message/event-ordering assumption that holds in single-threaded tests but not prod?
8. **Rollback / blast radius.** Failed and could not be cleanly reverted. Is this irreversible once shipped (new on-disk format, one-way migration, flag with no off switch)? Gated to limit blast radius, or 100% at once?
9. **Observability gap.** Happened, but found via customers not dashboards. New failure surface with no metric/log/alarm?
10. **Auth / input-trust boundary.** Became a security or data-exposure incident. Trusts input it should validate, widens a permission, logs a secret, or skips an authz check on a now-reachable path?

## Output discipline

For each cause: (a) the specific line/construct in the diff, (b) the prod failure it would produce, (c) a plausibility rank. The method's known weakness is inventing threats that are not real, so plausibility ranking is mandatory and the refutation pass will cull the speculative ones.
