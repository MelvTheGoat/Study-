# Production-Grade-Double-Entry-Ledger-API: System Design

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

## The problem, in 3 lines

Money is the hardest thing to get right in a backend: two payments at the same instant can overdraw an account, a retried request can move money twice, and an event published before a commit can announce a transfer that never happened.
This is a double-entry ledger and payments API (FastAPI, PostgreSQL 16, Redis 7) where correctness under concurrency, retries, partial failure and replay is **built into the database and proven by tests and benchmarks**.
Every transfer is balanced entries, the ledger is append-only, and every write is idempotent.

## Diagram

```mermaid
flowchart LR
    C[Client] -->|X-API-Key, Idempotency-Key| MW

    subgraph API["FastAPI (src/api)"]
        MW[Middleware: request_id, logging,<br/>auth (SHA-256 keys + scopes),<br/>rate limit (Redis sliding log, Lua)]
        IDEM[Idempotency ASGI middleware<br/>INSERT ... ON CONFLICT DO NOTHING]
        R[Routes: accounts, transfers,<br/>reversals, statements, trial balance]
        ERR[RFC 7807 problem+json]
    end

    subgraph CORE["Ledger core (src/ledger)"]
        SVC[Service: post balanced entries,<br/>update cached balance with delta]
        LOCK[Locking: pessimistic (default)<br/>or optimistic CAS]
    end

    subgraph PG["PostgreSQL 16 (the source of truth)"]
        T[(transactions)]
        E[(entries<br/>append-only triggers,<br/>deferred balance trigger)]
        A[(accounts<br/>balance cache, sign check,<br/>version)]
        IK[(idempotency_keys<br/>unique constraint)]
        OB[(outbox_events)]
        AU[(audit_log)]
    end

    subgraph ASYNC["Async"]
        REL[Outbox relay<br/>FOR UPDATE SKIP LOCKED]
        RS[(Redis Streams)]
        CON[Idempotent consumer<br/>SET NX on event_id]
    end

    MW --> IDEM --> R --> SVC --> LOCK --> A
    SVC --> T & E & OB & AU
    IDEM --> IK
    R --> ERR
    OB --> REL --> RS --> CON
    MW -.rate limit.-> RS
    REC[scripts/reconcile.py<br/>recompute balances from entries] -.-> PG
```

## Each part, and why it's there

| Part | Code | What it does | Why it's there |
|---|---|---|---|
| Money model | `src/ledger/models.py`, migration | `BIGINT` minor units, explicit currency on every amount, signed `amount`. Strict API rejects `"1050"` and `10.50`. | No float rounding, and no default currency. |
| Double-entry invariant | `migrations/versions/0001_initial_schema.py` | A **deferred constraint trigger** checks every transaction's entries sum to 0 and match the declared entry count. | Enforced even if app code is bypassed. The count check stops appending a "balanced" pair to history. |
| Append-only ledger | same | `BEFORE UPDATE/DELETE/TRUNCATE` triggers on `entries` and `transactions` raise. | Corrections are reversals. History can't be rewritten. |
| Currency safety | same | Composite FKs `(account_id, currency)` and `(transaction_id, currency)`. | Cross-currency entries are impossible, not just rejected. |
| Balance cache | `src/ledger/service.py` | `accounts.balance` updated with `balance = balance + :delta` in the same DB transaction. `?derived=true` recomputes. | O(1) reads (36× faster), and never stale. |
| Locking | `src/ledger/locking.py` | Pessimistic `SELECT ... FOR UPDATE`, one statement per account, in ascending id order (default). Optimistic `WHERE version = :expected` as an option. | Prevents the overdraft race and deadlocks. |
| Sign constraint | `ck_accounts_balance_sign` | A non-negative balance unless `allow_negative`. | The last line of defence if locking is wrong. |
| Idempotency | `src/api/idempotency.py` | Raw ASGI middleware. `INSERT ... ON CONFLICT DO NOTHING RETURNING id` claims the key. The claim is finalised inside the ledger transaction. Same payload → replay. Different payload → 422. In flight → 409 + Retry-After. | Retries are normal. Exactly-once effect. |
| Auth | `src/api/auth.py` | `lk_<env>_<body>` keys with an indexed prefix, stored as SHA-256, with scopes (`reversals:write` separate from `transfers:write`). | Fast lookup of high-entropy keys. Undoing money needs more privilege. |
| Rate limiting | `src/api/ratelimit.py` | Redis sliding-window log in one Lua call, per API key. **Fails open** with a circuit breaker. | Protects capacity. Correctness lives in Postgres. |
| Errors | `src/api/errors.py` | RFC 7807 problem+json on every path, including framework 404/405/422. | Consistent machine-readable errors. |
| Pagination | `src/api/pagination.py` | Keyset (cursor) pagination. | OFFSET double-counts rows while new ones arrive, and gets slower with depth. |
| Outbox | `src/events/outbox.py`, `relay.py`, `consumer.py` | The event row is written in the same commit. The relay uses `FOR UPDATE SKIP LOCKED`, publishes to Redis Streams, then marks it published. The consumer dedupes with `SET NX`. | No dual write. At-least-once with no loss. |
| Resilience | `src/resilience/*` | Circuit breaker, graceful shutdown (readiness flips first). | Degradation paths tested, not assumed. |
| Observability | `src/observability/*` | JSON logs with `request_id` via contextvars, Prometheus metrics. | Traceable requests. |
| Reconciliation | `scripts/reconcile.py` | Recomputes balances from entries and checks the trial balance. | Proves the cache is correct. |
| Benchmarks | `scripts/benchmark.py`, `locking_bench.py`, `index_write_cost.py`, `loadtest/` | Query plans, locking comparison, index write cost, locust load test. Results committed in `bench-results/`. | Evidence for every performance claim. |

## Tech stack

| Tool | What it's used for | Why this one |
|---|---|---|
| Python 3.12, FastAPI | API | Async, typed |
| SQLAlchemy (async) + asyncpg | DB access | Async Postgres |
| PostgreSQL 16 | Ledger store | Triggers, deferred constraints, row locks, SKIP LOCKED |
| Alembic | Migrations | Versioned schema with invariants |
| Redis 7 | Rate limiting, event stream, consumer dedupe | Fast, atomic Lua, Streams |
| Prometheus client | Metrics | Standard |
| pytest + hypothesis | Tests incl. property-based | Against **real** Postgres and Redis |
| locust | Load testing | Realistic mixed workload |
| Docker Compose | Local stack | API, Postgres, Redis, worker, migrations |
| GitHub Actions | CI | Service containers for Postgres and Redis |

## Data flow, step by step (`POST /v1/transfers`)

1. The middleware assigns a `request_id`, authenticates the API key (SHA-256 lookup by prefix), checks scopes and applies the rate limit.
2. **Idempotency:** `INSERT ... ON CONFLICT DO NOTHING`. If this request wins, it executes. If the same key and payload are done, replay the stored response. Different payload → 422. In flight → 409.
3. **Validate** strictly: integer amount, explicit currency, accounts exist, same currency.
4. **Begin a DB transaction.** Lock both accounts in ascending id order (`FOR UPDATE`, one statement each).
5. **Check funds.** Insert the `transaction` (with declared entry count) and two `entries` (−amount, +amount).
6. **Update balances** with `balance = balance + :delta`. The sign constraint is checked.
7. **Write** the `outbox_events` row and the audit record. Finalise the idempotency claim **in the same transaction**.
8. **COMMIT.** The deferred trigger verifies the entries sum to 0 and the count matches.
9. Cache the response body for replay (if this is lost, a replay reconstructs it from the committed transaction).
10. **Later:** the relay picks up the pending outbox rows (`SKIP LOCKED`), publishes to Redis Streams, and marks them published. Consumers dedupe on `event_id`.

## Trade-offs and limits

**Measured (README, with committed artefacts in `bench-results/`):**
- **Locking:** on a hot account, pessimistic runs at **144.8 tps** with 0 conflicts. Optimistic runs at 41.3 tps and sheds 46% of writes as conflicts (4,270 retries). Spread across 16 accounts: 206.5 vs 112.6 tps.
- **N+1 fix:** 101 queries in 17.83 ms → 1 query in 1.50 ms (11.9×).
- **Keyset vs OFFSET** at depth 5,000: 0.251 ms vs 4.386 ms.
- **Cached vs derived balance:** 0.209 ms vs 7.412 ms (36×).
- **Index `ix_entries_account_id_id`:** 70.78 ms → 0.173 ms (409×). An `entries(amount)` index was rejected (+14% write cost).
- **Load test** (4-vCPU box, everything co-located): it saturates at ~250–260 req/s. At 50 users, p99 is 1,400 ms with a hot account vs 360 ms without. 0 failures, and reconciliation passes after 17,697 transactions.

**Limits (from the repo):**
- Single-writer Postgres. No sharding, replicas or partitioning.
- Throughput not proven beyond ~250 req/s (a shared box).
- No FX (cross-currency refused).
- All-or-nothing reversals.
- Rate limits per key, not per endpoint.
- Outbox adds up to 500 ms delivery latency (polling).
- Unbounded `audit_log`.
- Idempotency buffers responses (256 KB cap).
- No distributed tracing.
- Bootstrap needs a Python one-liner.
- The only migration's downgrade is destructive.
- Hypothesis runs on shared DB state.

## What I'd change at 10x scale

- **Partition `entries`** by time or account range. Move the trial balance to a scheduled job or snapshot.
- **Read replicas** for statements and balances (with care over read-your-writes).
- **Shard by account** (or by tenant) when a single writer saturates. Keep transfers within a shard, or use a two-phase ledger pattern across shards.
- **Hot-account strategies:** sub-accounts or batching for very hot accounts (e.g. a platform's fee account).
- **`LISTEN/NOTIFY` + polling backstop** for lower event latency, or CDC (Debezium) from the WAL.
- **OpenTelemetry tracing**, per-endpoint rate limits, and an audit log retention policy.
- **An FX account pattern** with rate provenance.
