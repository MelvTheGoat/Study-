# Production-Grade-Double-Entry-Ledger-API: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

---

## 60-second pitch

> "I built a double-entry ledger and payments API on FastAPI, PostgreSQL and Redis, where correctness under concurrency and retries is enforced by the database and proven by tests. Money is BIGINT minor units with explicit currency. A deferred constraint trigger forces every transaction's entries to sum to zero. The ledger is append-only, and composite foreign keys make cross-currency entries impossible.
>
> For the overdraft race, accounts are row-locked in ascending id order. I benchmarked that against optimistic locking: on a hot account it's 145 vs 41 transfers per second, and optimistic rejected 46% of writes. Idempotency uses a unique-constraint claim finalised in the same transaction as the money, and events go through a transactional outbox. All 140 tests run against real Postgres and Redis. After a 17,697-transaction load test, reconciliation came out exactly zero."

---

## Questions and honest answers

### 1. "Why double-entry?"
Every movement is two or more entries summing to zero, so money is never created or destroyed, and any balance can be recomputed from its entries. It's the standard accounting model and makes errors detectable (the trial balance must be zero).

### 2. "Why enforce rules in the database instead of the service?"
Because the service isn't the only writer: migrations, scripts, other services and humans all touch the DB. A deferred trigger and append-only triggers can't be bypassed. My tests write raw SQL to prove the DB refuses unbalanced or edited entries.

### 3. "How do you prevent two transfers overdrawing an account?"
Lock the account rows with `SELECT ... FOR UPDATE` before checking funds, update with `balance = balance + delta`, and a sign constraint as the backstop. Under READ COMMITTED without locks, both would read 100 and both would write.

### 4. "Why pessimistic over optimistic?"
Measured: on a hot account, pessimistic 144.8 tps with 0 conflicts, optimistic 41.3 tps with 46% conflicts and 4,270 retries. Optimistic is fine when contention is rare, but payments have hot accounts by nature. A CI test asserts optimistic starves first under heavy contention.

### 5. "How do you avoid deadlocks?"
Always lock in ascending account-id order, one statement per row. A single `WHERE id = ANY(...) ORDER BY id FOR UPDATE` doesn't guarantee lock order in Postgres, so it can still deadlock.

### 6. "Why not SERIALIZABLE?"
It's correct, but predicate locks cost memory and CPU, aborts are frequent under contention, and every transaction has to use it. For a hot account, explicit row locks under READ COMMITTED are cheaper and more predictable.

### 7. "How does idempotency work?"
`INSERT ... ON CONFLICT DO NOTHING RETURNING id` on (api_key, endpoint, key). Exactly one concurrent request wins. The claim is finalised in the same DB transaction as the ledger write, so no crash window lets money move twice. Same payload replays, a different payload gives 422, in flight gives 409.

### 8. "Why not a Redis lock for idempotency?"
A lock can expire while the holder is still working, or disappear in a failover, and then two requests execute. The unique constraint lives with the data and can't disagree with it.

### 9. "How do you publish events reliably?"
A transactional outbox: the event row commits with the entries. A relay uses `FOR UPDATE SKIP LOCKED`, publishes to Redis Streams, then marks it published. A crash can cause a duplicate but never a loss. Consumers dedupe on `event_id` with `SET NX`.

### 10. "Why keyset pagination?"
OFFSET double-counts rows when new entries arrive between pages, which is fatal for statements, and it's O(offset): 4.39 ms vs 0.25 ms at depth 5,000.

### 11. "How does it perform?"
On a shared 4-vCPU box it saturates around 250–260 req/s, and adding users only raises latency (Little's law). The hot account doesn't cut throughput. It raises p99 (1,800 ms vs 390 ms for transfers). No failures, and reconciliation passes.

### 12. "How would you scale it?"
Partition `entries`, add read replicas for statements, shard by account or tenant, split hot accounts into sub-accounts, use CDC or NOTIFY with polling for events, and trace with OpenTelemetry.

### 13. "Why SHA-256 for API keys, not bcrypt/Argon2?"
The keys are 240-bit random secrets, not human passwords, so there's nothing to brute-force slowly. A slow KDF would just give attackers a CPU-exhaustion lever on an unauthenticated path.

### 14. "What happens if Redis goes down?"
Rate limiting fails open, readiness reports "degraded", and money endpoints keep working. Authentication never fails open. If Postgres is down, it fails closed with a 503, because the database is the ledger.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Single Postgres | "Doesn't scale." | "Not yet. Partitioning and sharding are the plan. Correctness first." |
| 250 req/s | "That's low." | "It's one shared 4-vCPU box running everything. The saturation shape is real. The ceiling is unknown." |
| No FX | "Real payments need FX." | "Cross-currency is refused as an honest placeholder. FX needs an FX account pattern with rate provenance." |
| Outbox latency | "Up to 500 ms?" | "Polling is durable. NOTIFY would still need polling as a backstop." |
| Test count | "README says 138." | "I ran 140 against a fresh Postgres 16 and Redis. They all passed." |
