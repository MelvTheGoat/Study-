# Production-Grade-Double-Entry-Ledger-API: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

---

## 1. Double-entry accounting

**Simple explanation:** every transaction is at least two entries that sum to zero, so money is conserved.

**In this project:** signed entries, a deferred balance trigger, the trial balance, reversals, account types.

**Also be ready to explain:** debits vs credits, the account types (asset, liability, equity, revenue, expense) and their normal balances, the trial balance, reversal vs adjustment, and suspense accounts.

---

## 2. Database integrity (constraints and triggers)

**Simple explanation:** make the database refuse invalid data.

**In this project:** a deferred constraint trigger, append-only triggers, check constraints, composite foreign keys, unique constraints.

**Also be ready to explain:** deferred vs immediate constraints, statement vs row triggers, and why app-level validation alone isn't enough.

---

## 3. Concurrency control and isolation levels

**Simple explanation:** stop simultaneous transactions from corrupting each other.

**In this project:** READ COMMITTED plus `SELECT FOR UPDATE`, optimistic version CAS, ordered locking, delta updates.

**Also be ready to explain:** dirty and non-repeatable reads, phantoms, lost updates, write skew, READ COMMITTED vs REPEATABLE READ vs SERIALIZABLE (SSI) in Postgres, EvalPlanQual, and deadlock conditions and prevention.

---

## 4. Idempotency

**Simple explanation:** doing the same request twice has the same effect as doing it once.

**In this project:** Idempotency-Key, a unique-constraint claim, same-transaction finalisation, replay reconstruction.

**Also be ready to explain:** at-least-once delivery, idempotency key design (scope, TTL), and why not to rely on distributed locks.

---

## 5. Reliable messaging (transactional outbox)

**Simple explanation:** write the event in the same database commit as the data, then publish it separately.

**In this project:** the outbox table, a SKIP LOCKED relay, Redis Streams, idempotent consumers.

**Also be ready to explain:** the dual-write problem, exactly-once vs at-least-once, CDC (Debezium), inbox pattern, and sagas.

---

## 6. API design

**Simple explanation:** clear, safe, consistent HTTP endpoints.

**In this project:** REST under `/v1`, RFC 7807 errors, keyset pagination, scopes, strict validation, OpenAPI docs.

**Also be ready to explain:** status codes (409 vs 422), versioning, cursor design, and least-privilege scopes.

---

## 7. Query performance and indexing

**Simple explanation:** make queries fast without slowing writes too much.

**In this project:** fixing an N+1, EXPLAIN ANALYZE, a composite index, a partial index, rejecting an index over write cost, keyset vs OFFSET.

**Also be ready to explain:** B-tree indexes, index-only scans, the planner choosing sequential scans, partial indexes, and write amplification.

---

## 8. Performance testing and capacity

**Simple explanation:** measure how much load the system handles and where it breaks.

**In this project:** locust load tests, saturation at ~250 req/s, Little's law, connection pool sizing, hot-row contention.

**Also be ready to explain:** throughput vs latency, p50/p95/p99, Little's law (L = λW), and pool sizing (~2 × cores).

---

## 9. Resilience and observability

**Simple explanation:** keep working (or fail safely) when dependencies break, and see what's happening.

**In this project:** fail closed on Postgres and open on the rate limiter, a circuit breaker, graceful shutdown, JSON logs with request_id, Prometheus metrics.

**Also be ready to explain:** circuit breaker states, readiness vs liveness probes, and structured logging and tracing.

---

## 10. Security basics for APIs

**In this project:** API key format with a prefix lookup, SHA-256 storage, scopes, a sliding-window rate limiter in Lua, and auth that never fails open.

**Also be ready to explain:** hashing vs KDFs (and when each fits), rate-limiting algorithms (fixed window, sliding log, token bucket), and timing-safe comparison.

---

## 11. Testing with real dependencies

**Simple explanation:** test against the real database when the correctness lives there.

**In this project:** 140 tests against real Postgres and Redis, raw-SQL bypass tests, hypothesis property tests, concurrency tests with separate connections.

**Also be ready to explain:** testcontainers, why mocks can't prove DB behaviour, and property-based testing.
