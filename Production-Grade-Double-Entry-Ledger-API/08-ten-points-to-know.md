# Production-Grade-Double-Entry-Ledger-API: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

1. **A double-entry ledger + payments API: Python 3.12, FastAPI, PostgreSQL 16, Redis 7.**
   *Why it matters:* the stack and the purpose in one line.

2. **Money is BIGINT minor units with explicit currency. Strict input (no strings or decimals, no default currency).**
   *Why it matters:* exact money.

3. **A deferred constraint trigger enforces sum = 0 and the declared entry count per transaction.**
   *Why it matters:* double-entry is guaranteed by the DB.

4. **Append-only: triggers block UPDATE/DELETE/TRUNCATE. Corrections are reversals. Composite FKs make cross-currency impossible.**
   *Why it matters:* history can't be rewritten.

5. **Overdraft race fixed by ordered `FOR UPDATE` row locks (one statement each), delta updates and a sign constraint.**
   *Why it matters:* the core concurrency answer.

6. **Pessimistic 144.8 tps vs optimistic 41.3 tps on a hot account (46% conflicts).**
   *Why it matters:* a measured, defensible design choice.

7. **Idempotency: `ON CONFLICT DO NOTHING` claim, finalised in the same transaction. Replay / 422 / 409.**
   *Why it matters:* retries never double-move money.

8. **Transactional outbox + SKIP LOCKED relay + `event_id` dedupe. Duplicates are possible, loss isn't.**
   *Why it matters:* no dual write.

9. **Performance: N+1 fix 11.9×, keyset vs OFFSET 17× at depth 5k, cached balance 36×. Saturation ~250 req/s on 4 vCPU. Hot account raises p99, not throughput.**
   *Why it matters:* evidence-based performance talk.

10. **140 tests pass against real Postgres + Redis (my run). Property-based conservation test. Reconciliation exactly zero after 17,697 transactions.**
    *Why it matters:* proof, not claims.
