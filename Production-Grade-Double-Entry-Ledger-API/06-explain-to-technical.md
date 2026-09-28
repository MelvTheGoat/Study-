# Production-Grade-Double-Entry-Ledger-API: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

---

## Summary

A double-entry ledger and payments API (Python 3.12, FastAPI, async SQLAlchemy/asyncpg, PostgreSQL 16, Redis 7). Invariants are enforced in the database (a deferred balance trigger with an entry count, append-only triggers, a sign check, composite currency FKs). Pessimistic ordered row locking by default, with optimistic CAS as an option. Unique-constraint idempotency finalised inside the ledger transaction. Transactional outbox with a SKIP LOCKED relay to Redis Streams and idempotent consumers. RFC 7807 errors, keyset pagination, SHA-256 API keys with scopes, a Redis sliding-log rate limiter that fails open, a circuit breaker, graceful shutdown, JSON logs and Prometheus metrics. **I ran the suite against a fresh Postgres 16 + Redis: 140 passed (~59 s).** Benchmark artefacts are committed. ~6,000 lines, 12 commits on 8 Aug 2026.

## Architecture

```
src/api/app.py, middleware.py   app factory, request_id, logging, metrics
src/api/auth.py                 lk_<env>_<body>, prefix lookup, SHA-256, scopes
src/api/ratelimit.py            Redis ZSET sliding log, one Lua call, fail-open + breaker
src/api/idempotency.py          raw ASGI middleware, ON CONFLICT claim, replay/422/409
src/api/pagination.py           keyset cursors
src/api/errors.py               RFC 7807 everywhere
src/api/routes/*                accounts, transfers, reversals, trial balance, admin, ops
src/ledger/models.py, enums.py  schema models
src/ledger/locking.py           pessimistic (ordered FOR UPDATE per row) / optimistic (version CAS)
src/ledger/service.py           postings, balances (delta), statements (join), reversals
src/events/outbox.py, relay.py  outbox rows, SKIP LOCKED relay -> Redis Streams
src/events/consumer.py          SET NX dedupe on event_id
src/resilience/*                circuit breaker, graceful shutdown
src/observability/*             contextvars logging, Prometheus metrics
migrations/0001                 tables, triggers, constraints, indexes
scripts/                        reconcile, seed, benchmark, locking_bench, index_write_cost
loadtest/                       locust
```

## Key decisions (16 ADRs in DECISIONS.md)

| ADR | Decision | Why |
|---|---|---|
| 001 | Signed BIGINT minor units | Exact, fast, no floats |
| 002 | Signed `amount` (not amount + direction) | Sum = 0 is a simple check |
| 003 | Invariants in the DB | Can't be bypassed |
| 004 | `accounts.balance` is a cache, not the truth | O(1) reads, reconcilable |
| 005 | Pessimistic locking default | Measured 3.5× on hot rows |
| 006 | Locks one statement at a time, canonical order | Multi-row FOR UPDATE order isn't guaranteed |
| 007 | Unique constraint for idempotency exclusion | No lock expiry or failover risk |
| 008 | SHA-256 API keys, not Argon2 | 240-bit random secrets. A KDF would be a CPU-DoS lever. |
| 009 | Rate limit fails open, auth never does | Capacity ≠ correctness |
| 010 | Reject cross-currency | Honest placeholder for FX |
| 011 | Keyset pagination | OFFSET double-counts and is O(offset) |
| 012 | Pool 20 + 10 per worker | Postgres slows with too many backends |
| 013 | Transactional outbox | No dual write |
| 014 | Polling, not LISTEN/NOTIFY | NOTIFY isn't durable |
| 015 | Idempotency as ASGI middleware buffering the response | Replay byte-for-byte |
| 016 | Idempotency keys scoped per API key | No cross-tenant collisions |

## Concurrency details

- **READ COMMITTED** plus row locks. `UPDATE ... WHERE version = :v` is sound under RC because of EvalPlanQual re-checking.
- Why not **REPEATABLE READ** (it turns races into 40001 errors) or **SERIALIZABLE** (SSI predicate-lock overhead, false-positive aborts, and every writer must use it).
- Balance updates as deltas. The sign constraint is the backstop.

## How it's tested

- **140 tests passed in my run** against a real Postgres 16 and Redis (the README lists 138: invariants 31, concurrency 17, idempotency 15, API 37, outbox 11, resilience 17, reconciliation 6, property 4).
- Raw-SQL tests bypass the service to prove the DB rejects bad writes.
- Hypothesis checks conservation after every step.
- Coverage 90% (README). The gaps are process entry points.
- **Benchmarks (committed `bench-results/`):** locking (pessimistic hot 144.8 tps vs optimistic 41.3 with 46% conflicts), N+1 17.83 → 1.50 ms, keyset 0.251 vs OFFSET 4.386 ms at depth 5k, cached 0.209 vs derived 7.412 ms, statement index 70.78 → 0.173 ms, rejected `entries(amount)` index (+14% writes), load test ~250–260 req/s ceiling on 4 vCPU, hot-account p99 1,400 ms vs 360 ms without (aggregate), 0 failures.

## Known weaknesses

- Single-writer Postgres. No partitioning, replicas or sharding.
- Throughput ceiling unknown beyond the shared 4-vCPU box.
- No FX. All-or-nothing reversals. No adjustment endpoint.
- Per-key (not per-endpoint) rate limiting.
- Outbox polling latency (≤500 ms).
- Unbounded audit log.
- 256 KB idempotency response buffer.
- No distributed tracing.
- A destructive first-migration downgrade.
- Hypothesis examples share DB state.
- Bootstrap needs a Python one-liner.
