# Production-Grade-Double-Entry-Ledger-API: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

This is a classic backend interview question ("design a payments ledger"). Here's how to walk through it using this project.

---

## Step 1: Requirements (3 min)

One line:
> "An API to create accounts and move money between them, where money is never created, lost, double-spent or rewritten, even under concurrency and retries."

**Functional**
1. Create accounts (with currency, type, and whether negative balances are allowed).
2. Transfer money between accounts.
3. Reverse a transaction.
4. Read balances and statements, and a trial balance.
5. Emit events for downstream systems.

**Non-functional (these are the design)**
- **Correctness:** double entry (sum = 0), no overdraft, append-only, one currency per transaction.
- **Idempotency:** retries never double-move money.
- **Consistency under concurrency:** no lost updates or deadlocks.
- **Reliable events:** no phantom or lost events.
- **Auditability**, and **graceful degradation** when optional dependencies fail.

---

## Step 2: Numbers (2 min)

| Thing | Number |
|---|---|
| Benchmark dataset | 100k accounts, 600k transactions, 1.2M entries |
| Hot-account skew | 0.1% of accounts get 60% of traffic |
| Load-test ceiling | ~250–260 req/s on one 4-vCPU box |
| Connection pool | 20 + 10 overflow per worker |
| Idempotency response cap | 256 KB |

**Say:** "Correctness first. Throughput comes from scaling out later. The hot account is the real bottleneck."

---

## Step 3: High-level boxes (3 min)

```
Client --(API key, Idempotency-Key)--> [FastAPI: auth, rate limit (Redis), idempotency middleware]
     --> [Ledger service: lock accounts, insert txn + entries, update balances, outbox row]
     --> [PostgreSQL: triggers + constraints enforce invariants]
[Outbox relay (SKIP LOCKED)] --> [Redis Streams] --> [Idempotent consumers]
[Reconcile job] recomputes balances from entries
```

---

## Step 4: Deep dive (12 min)

### 4a. Data model
- `accounts(id, currency, balance BIGINT, version, allow_negative)`
- `transactions(id, currency, kind, entry_count)`
- `entries(id, transaction_id, account_id, currency, amount BIGINT signed)`
- `idempotency_keys`, `outbox_events`, `audit_log`, `api_keys`
- **Invariants in the DB:** a deferred trigger (sum = 0, count matches), append-only triggers, a sign check, composite currency FKs.

### 4b. The overdraft race
- Two concurrent 60s against 100 under READ COMMITTED both pass the check.
- Fix: **lock the rows** (`SELECT ... FOR UPDATE`), and update with `balance = balance + delta`. The constraint is the backstop.
- **Deadlock:** always lock in ascending account-id order, one statement per row.
- Compare: optimistic (`WHERE version = :v`) sheds 46% on a hot row. SERIALIZABLE costs predicate locks and retries.

### 4c. Idempotency
- Claim: `INSERT ... ON CONFLICT DO NOTHING RETURNING id`.
- Finalise the claim in the **same transaction** as the ledger write.
- Replay returns the stored response. A different payload → 422. In flight → 409.

### 4d. Events
- The outbox row is written in the same commit (no dual write).
- The relay uses `FOR UPDATE SKIP LOCKED`, publishes, then marks it published. That means duplicates are possible, but loss isn't.
- Consumers dedupe on `event_id`.

### 4e. Reads
- A cached balance (O(1)) vs a derived SUM (O(n)), both exposed.
- Keyset pagination for statements.
- An index on `(account_id, id)` for statements.

### 4f. Failure handling
- Postgres down → fail closed (503).
- Redis down → rate limiting fails open. Auth never does.
- Graceful shutdown flips readiness first.

---

## Step 5: Bottlenecks (3 min)

1. **Hot accounts:** lock queues raise tail latency (p99 1,800 ms vs 390 ms without a hot account).
2. **CPU saturation** at ~250 req/s on the test box.
3. **Single-writer Postgres** eventually.
4. **Full scans** (trial balance) grow with entries.

---

## Step 6: Trade-offs (3 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Invariants in DB | App-only checks | Can't be bypassed | Harder migrations |
| Pessimistic locking | Optimistic / SERIALIZABLE | 3.5× throughput on hot rows, no shedding | Queueing tail latency |
| Unique-constraint idempotency | Redis lock | No expiry or failover double execution | Extra table |
| Cached balance | Derived only | 36× faster reads | Must reconcile |
| Outbox + polling | Publish in handler / NOTIFY | No dual write, durable | Up to 500 ms latency |
| Keyset pagination | OFFSET | Stable and fast | No random page access |
| SHA-256 API keys | Argon2 | High-entropy keys, avoids a CPU DoS | Not suitable for passwords |
