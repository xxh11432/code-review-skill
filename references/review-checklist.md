# Review checklist

Questions to ask inside each pass. Not every question applies to every diff — answer the ones the change actually touches.

## Correctness

- Empty, null, zero, negative, maximum, and duplicate inputs handled?
- Boundaries correct (off-by-one, inclusive/exclusive ranges, timezone and DST, leap years)?
- Early returns and `continue`/`break` paths release everything they acquired?
- Nullable values dereferenced only after a guard that the compiler/runtime actually enforces?
- State machines and enums: every branch handled, including the default?
- Mutations of shared objects visible to other callers?
- Floating-point money and time arithmetic replaced by exact/integer types?

## Security

- Every external input validated (HTTP params, headers, file uploads, queue payloads, CLI args)?
- Secrets, tokens, and credentials read from config or a secret store, never hardcoded or logged?
- Authorization checked at the resource, not just at the route?
- SQL/ORM queries parameterized; no string concatenation of user input?
- Output escaped per context (HTML, shell, logs, CSV) to prevent injection?
- Deserialization restricted to expected types; no arbitrary object instantiation?
- Paths from user input normalized and confined to a base directory?
- TLS verification not disabled, certificate checks not bypassed?
- New dependencies pinned, maintained, and not pulling in a wider attack surface than needed?

## Data and migrations

- Schema change applied in an order that old and new code can both tolerate?
- New NOT NULL column has a backfill or a safe default?
- Index added for every new high-cardinality query path, and created without a long lock?
- Destructive changes (drop/rename) staged behind a deprecation step?
- Idempotent writes: repeated execution produces the same result?
- Rollback path exists and does not lose data created after deploy?

## Concurrency and resources

- Locks acquired in a consistent order; no path acquires two locks in opposite orders?
- Transactions as short as possible; external calls not made while holding a lock?
- Pools, channels, iterators, streams, and file handles closed on every path, including exceptions?
- Retries bounded, with backoff and jitter; non-idempotent operations not retried?
- Shared mutable state either immutable, confined, or protected?
- Background tasks and timers cancelled on shutdown?

## Performance

- No query inside a loop where the set could be fetched once (N+1)?
- No unbounded `SELECT`, `find_all`, or list endpoint without pagination?
- Hot paths free of repeated serialization, regex compilation, or large allocations?
- Caching added with a defined invalidation strategy, not just a TTL guess?
- Large payloads streamed rather than buffered whole?
- Complexity of new loops justified by the data sizes they will actually see?

## Error handling

- Exceptions caught only where they can be handled; no bare `catch {}` that hides failures?
- Errors propagated with context (operation, identifier) without leaking secrets?
- Partial failures leave consistent state, or the operation is retried as a whole?
- User-facing messages distinguish retryable from permanent failures?
- Timeouts set on every network call?

## API and contracts

- Breaking changes to public signatures, response fields, error codes, or defaults called out explicitly?
- Serialized formats versioned; new fields optional for old readers?
- Config keys added with defaults and documented; removed keys still tolerated for one release?
- Feature flagged if the behavior change is risky or gradual?
- Callers in other repositories or services updated or given a migration window?

## Tests

- New behavior has a test that fails when the change is reverted?
- Failure paths and boundary values tested, not just the happy path?
- Tests deterministic: no sleeps, no real clock, no shared external state, no order dependence?
- Test assertions check behavior, not implementation details that will churn?
- Existing tests updated where semantics intentionally changed — and the change is called out?

## Maintainability (diff-local only)

- New abstraction earns its keep; no single-use wrapper or speculative generality?
- Duplication introduced by this diff merged into the existing helper?
- Dead code, unused imports, commented-out blocks, and debug output removed?
- Names describe intent; booleans read as predicates?
- Nesting reduced where the diff introduced a deep pyramid?
- Comments explain why, not what; stale comments updated?

## Language hotspots

- **Python**: mutable default arguments, late-binding closures, `except Exception` swallowing, blocking calls in async code, `datetime.now()` without timezone.
- **JavaScript / TypeScript**: floating `await` without `try`, unhandled promise rejections, `any` at a module boundary, stale closures in hooks, `==` coercion, missing cleanup in effects.
- **Java / Kotlin**: resource leaks without try-with-resources, `equals`/`hashCode` pairs, mutable shared fields, swallowed `InterruptedException`, nullable platform types.
- **Go**: ignored `error` values, goroutine leaks, `defer` inside loops, unclosed `response.Body`, data races on shared maps.
- **SQL**: missing index for new predicates, `NULL` semantics in comparisons, implicit casts that prevent index use, migrations that rewrite a large table in place.
