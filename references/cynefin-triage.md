# Cynefin triage (sets review depth)

Source: Snowden, Cynefin framework (IBM Systems Journal 2003; HBR 2007). Classify the change by how knowable its consequences are, then pick depth. Runs FIRST, before line-by-line reading.

**Operational rule: start in Disorder, score the signals, escalate on the single strongest signal. Do NOT average down.** One real Complex signal (a lock change buried in a tiny diff) outranks ten cosmetic lines. The cheap-looking change sitting in Complex territory is exactly what slips a checklist and breaks prod.

## Tiers and depth

| Tier | Kind of change | Lenses to run |
|---|---|---|
| **Clear** | rename, format, comment, dead-code removal, dependency bump, generated code, config value in a well-trodden file | Focused dimension agents only (staleness, dead code, silent failure, guidance compliance). No framework fan-out. Say it was light. |
| **Complicated** | normal feature/bugfix in one well-understood module, local effects, single owner | Starred framework lenses (STAMP, Pre-mortem, Inversion) + all focused dimension agents. 2 skeptics per finding. |
| **Complex** | effects only show in hindsight (see signals below) | ALL framework lenses (STAMP, Pre-mortem, Inversion, Data-flow, Contract, Omission, Concurrency) + all focused dimension agents. 3 skeptics per finding. Reading the diff is NOT enough; probe the package, callers, and types. Treat the change as a hypothesis, not a known-correct edit. |
| **Chaotic** | active incident hotfix under fire | Review only blast radius + reversibility. Defer the rest. Say so. |

## Escalate to COMPLEX on ANY one of these signals

1. Touches a public API, exported interface, or wire/serialization format (schema, protobuf, event payload, DB column contract).
2. Changes concurrency, locking, async ordering, or threading. **(Also triggers the Concurrency lens regardless of final tier.)**
3. Alters a state machine, lifecycle, or transition logic.
4. Modifies a shared / high-fan-in dependency or library boundary.
5. Changes error handling, retry, timeout, backoff, or idempotency.
6. Involves a migration or backward/forward-compatibility concern (rolling deploy, version skew).
7. Crosses a security or trust boundary (authn/authz, input validation, secrets, deserialization, privilege).
8. Has distributed or cross-service effects (network call added/removed, ordering across services, cache invalidation, partial failure).
9. Touches money, quotas, rate limits, or correctness-critical numeric logic where a wrong answer is silent.
10. Large blast radius / low reversibility (hard to roll back, no feature flag, fans across many packages).
11. **New: touches data flow across a trust boundary** (user input reaching a query, external data reaching an auth decision, deserialized payload reaching business logic without re-validation).
12. **New: alters or breaks an implicit contract** (changes return type semantics, alters ordering guarantees, modifies what callers can assume even if the type signature stays the same).

## Stay CLEAR only when ALL hold

- Purely local edit, no caller-visible behavior change.
- No interface/schema/wire-format change.
- No concurrency, state, error/retry, or security surface touched.
- Single owning module, low fan-in, covered by existing tests.
- Trivially reversible.
- No data flow across trust boundaries.
- No implicit contract changes.

**Complicated** is the default middle: real logic, stays inside one well-understood module, clear local cause and effect, trips none of the Complex signals.

## Concurrency auto-trigger

If the diff contains ANY of these tokens/patterns, the Concurrency lens fires regardless of final tier: `mutex`, `lock`, `unlock`, `RwLock`, `atomic`, `Atomic`, `sync.`, `channel`, `Chan`, `select {`, `async`, `await`, `spawn`, `thread::`, `tokio::`, `goroutine`, `go func`, `pthread`, `synchronized`, `volatile`, `Arc<`, `Rc<`, `RefCell`, `Cell`, `unsafe impl Send`, `unsafe impl Sync`, `@synchronized`, `DispatchQueue`, `semaphore`, `condvar`, `notify_`, `wait(`, `CompletableFuture`, `CountDownLatch`, `ConcurrentHashMap`, `par_iter`, `rayon::`.

## Size-based depth adjustment

- If the diff is >500 lines of actual logic (not generated/vendor/test), escalate one tier (Clear->Complicated, Complicated->Complex). Large diffs hide interactions.
- If the diff touches >5 files across >2 directories, escalate one tier. Cross-cutting changes have emergent effects.
- These are floor raises, not overrides. A Complex signal still wins regardless of size.
