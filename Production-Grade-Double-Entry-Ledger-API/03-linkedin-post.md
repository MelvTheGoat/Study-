# Production-Grade-Double-Entry-Ledger-API: LinkedIn Post

*About 170 words. Copy from the line below.*

---

Alice has 100. Two transfers of 60 arrive at the same instant. Under Postgres's default isolation, both can succeed.

I built a double-entry ledger API (FastAPI, PostgreSQL, Redis) where that can't happen, and where the proof is tests and benchmarks, not a README claim.

The rules live in the database, not just the code:
- entries in every transaction must sum to exactly zero (a deferred constraint trigger)
- the ledger is append-only, so corrections are reversals
- cross-currency entries are impossible through composite foreign keys

The interesting bit: I benchmarked the two ways to stop the overdraft race. On a hot account, pessimistic row locking did 145 transfers/sec. Optimistic locking managed 41 and rejected 46% of writes as conflicts.

Retries are safe too. An Idempotency-Key is claimed with a unique constraint, and the claim commits in the same transaction as the money. Events use a transactional outbox, so no phantom or lost events.

A load test: 17,697 transfers under contention, and reconciliation came out exactly zero.

https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

#Backend #PostgreSQL #Fintech #Python
