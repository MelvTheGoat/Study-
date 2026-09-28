# Money is the hardest thing in a backend: building a ledger that proves it's correct

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

## Why I built it

Plenty of portfolio APIs show CRUD endpoints. That proves very little. The hard part of a payments backend isn't storing rows. It's staying correct when:

- two requests hit the same account at the same instant,
- a client times out and retries a transfer it can't tell succeeded,
- the process dies halfway between moving money and telling another system about it.

I wanted to build a double-entry ledger API (Python 3.12, FastAPI, PostgreSQL 16, Redis 7) where correctness under concurrency, retries, partial failure and replay is **built in and demonstrated**, with a test or a benchmark behind every claim, including the numbers that came out badly.

## The rules, enforced by the database

Application code gets bypassed: by the next migration script, a new service, or someone with `psql` during an incident. So the non-negotiable rules live in PostgreSQL.

**1. Money is an integer.** `BIGINT` in minor units, and currency is explicit on every amount. `1050` means USD 10.50. The API rejects `"1050"` and `10.50` rather than guessing, and there's no default currency.

**2. Every transaction balances.** At least two entries summing to exactly zero, enforced by a *deferred* constraint trigger that runs at commit time, after all the entries are in:

```sql
CREATE CONSTRAINT TRIGGER entries_balance_check
AFTER INSERT ON entries DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW EXECUTE FUNCTION ledger_assert_transaction_balanced();
```

It also checks the entry count the transaction declared up front. Otherwise someone could later append a *balanced* pair to an old transaction and quietly rewrite history.

**3. The ledger is append-only.** Triggers on `entries` and `transactions` refuse every `UPDATE`, `DELETE` and `TRUNCATE`. Mistakes are fixed by posting a reversal.

**Bonus:** composite foreign keys from entries to `(account_id, currency)` and `(transaction_id, currency)` make cross-currency entries *impossible*, not just rejected.

## The race that matters

> Alice has 100. Two transfers of 60 arrive at the same instant. Either alone succeeds. Together, they overdraw her by 20.

Under PostgreSQL's default isolation (READ COMMITTED), both transactions read `balance = 100`, both decide `60 ≤ 100`, and both write. The problem isn't the read. It's the gap between the read and the write.

I used three layers of defence, deliberately redundant:

1. **Locking:** serialise the read-then-write, either by locking the row or by detecting that it changed.
2. **Arithmetic:** balance updates are `balance = balance + :delta`, never `balance = :computed`, so even a lost update leaves the sum right.
3. **A constraint:** `ck_accounts_balance_sign`. If the first two layers fail, the write fails instead of corrupting the ledger.

### Avoiding deadlock

A transfer touches two rows, so A→B and B→A at the same time is the textbook deadlock. The fix is **ordering**: every writer locks accounts in ascending id order, so a cycle can't form. And each account is locked by its own statement:

```python
async def lock_accounts_pessimistic(...):
    """Take row locks on the given accounts in canonical order.

    Each account is locked by its own statement rather than by a single
    ``WHERE id = ANY(...) ORDER BY id FOR UPDATE``. ... PostgreSQL does not
    promise that a multi-row ``SELECT ... FOR UPDATE`` locks rows in
    ``ORDER BY`` sequence ...
```

It's one extra round trip per account, and worth it, because the single-statement version quietly throws away the deadlock protection the ordering was for.

### Pessimistic vs optimistic, measured

I made the locking strategy pluggable so I could measure it rather than argue about it. 32 workers × 8 transfers, 3 repeats:

| Scenario | Strategy | Throughput | Conflicts | Retries |
|---|---|---|---|---|
| Hot single account | **pessimistic** | **144.8 tps** | 0% | 0 |
| | optimistic | 41.3 tps | **46%** | 4,270 |
| Spread over 16 accounts | **pessimistic** | **206.5 tps** | 0% | 0 |
| | optimistic | 112.6 tps | 6% | 1,534 |

Pessimistic wins, and the gap grows with contention. Optimistic sheds almost half its writes on a hot row, at roughly ten attempts per success. Its p99 latency *looks* better (806 ms vs 1,244 ms), but that's survivorship bias: it gives up on the requests that would have been slow. For a payments ledger, where hot accounts are normal, pessimistic is the default. A test asserts this relationship, so a future change that reverses it fails CI.

The headline concurrency test funds Alice with exactly enough for half of N simultaneous transfers, fires all N at once on separate connections, and checks that exactly N/2 succeed, Alice ends at exactly 0, Bob gets exactly the right amount, and the cached balance matches a fresh `SUM` of the entries. It runs at N=25 and N=50, under both strategies.

## Retries without double-spending

A client whose request times out can't know whether the transfer happened. The only safe thing it can do is retry, and the only safe thing the server can do is make retrying free. So every write needs an `Idempotency-Key`:

| Case | Response |
|---|---|
| Same key, same payload | The original response, byte for byte. Nothing new is created. |
| Same key, different payload | 422: the client has a bug |
| Same key, still in progress | 409 + Retry-After |

The mutual exclusion is a **unique constraint**, not a lock:

```sql
INSERT INTO idempotency_keys (...) VALUES (...)
ON CONFLICT (api_key_id, endpoint, idempotency_key) DO NOTHING
RETURNING id;
```

Exactly one of N identical simultaneous requests gets a row back and executes. Not a Redis lock, because a lock can expire while its holder is still working, or vanish in a failover, and then you have two executors.

The claim is finalised **inside the same database transaction as the money movement**. If the process crashes before commit, nothing happened and a retry runs again. If it crashes after commit, the claim is already marked done, and a replay rebuilds the response from the committed transaction. There's no window where money moved and a replay would move it again.

## Events without dual writes

Publishing an event from the request handler is a bug whichever way round you do it. Publish then commit, and a failed commit leaves a phantom event. Commit then publish, and a crash in between loses the event forever. Two separate systems can't commit atomically without distributed transactions.

So I used a **transactional outbox**. The event is a row written in the same commit as the entries. A separate relay picks up pending rows with `FOR UPDATE SKIP LOCKED` (so several relays can run side by side), publishes to Redis Streams, then marks them published. If it crashes after publishing but before marking, the event is published again: a duplicate, not a loss. Consumers dedupe on `event_id`. A duplicate is fixable. A lost event isn't.

## Performance, including what didn't work

On a skewed dataset of 100,000 accounts and 1.2 million entries (0.1% of accounts get 60% of traffic):

- **N+1 found and fixed:** a statement page went from 101 queries (17.83 ms) to 1 join (1.50 ms), and a test checks both give identical results.
- **Keyset vs OFFSET pagination:** 0.25 ms vs 4.39 ms at depth 5,000. And OFFSET double-counts rows when new entries arrive between pages, which is fatal for a statement.
- **Cached vs derived balance:** 0.21 ms vs 7.41 ms. The cache is written in the same transaction, so it can't lag, and a reconcile script proves it.
- **An index I rejected:** `entries(amount)` made an offline report 4.5× faster but slowed every insert by about 14% on an append-only table.

A load test with locust (30% of transfers hitting one hot account) saturated around 250–260 requests/sec on a shared 4-vCPU box. The interesting part: removing the hot account didn't change throughput, but cut transfer p99 from 1,800 ms to 390 ms. Lock contention costs tail latency, not throughput. And after 17,697 transactions under load, reconciliation passed with a trial balance of exactly zero.

## Testing against the real thing

The tests all run against **real PostgreSQL and Redis**, with no mocked database, because the correctness lives in triggers, row locks and unique constraints that a mock can't prove. Several tests deliberately bypass the service and write raw SQL. A property-based test checks that total money is conserved after *every* step of a random sequence of transfers. When I ran the suite myself against a fresh local Postgres 16 and Redis, **140 tests passed**.

## What I learned

- **Put invariants where they can't be bypassed.**
- **Measure the locking strategy.** Intuition says optimistic is faster, but on hot rows it wasn't.
- **Idempotency is a database problem**, not a cache problem.
- **Never dual-write.** Use an outbox.
- **Report the bad numbers too.** The saturation curve and the rejected index taught me more than the wins.

## What's next

Partition `entries`, add read replicas, shard by account for scale, add FX with rate provenance, OpenTelemetry tracing, per-endpoint rate limits, and a retention policy for the audit log.
