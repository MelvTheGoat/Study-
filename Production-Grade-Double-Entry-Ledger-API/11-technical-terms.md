# Double-Entry Ledger API: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms are grouped by topic: accounting basics, the database, handling many requests at once, safe retries, the API, sending events, and proving it works.

---

## 1. Accounting Basics

### Ledger
**What it means:** the permanent record of every money movement.

**Why it's needed here:** it's the heart of the system. Balances, statements and reports all come from it.

### Double-entry bookkeeping
**What it means:** every transfer is recorded twice: money out of one account and money in to another. The two always add up to zero.

**Why it's needed here:** it makes it impossible for money to appear or vanish. If the entries don't add to zero, something is wrong.

### Entry
**What it means:** one line in the ledger: an account and an amount (negative for money out, positive for money in).

**Why it's needed here:** a transfer is made of entries. Using one signed amount makes "adds up to zero" a simple check.

### Minor units
**What it means:** the smallest unit of a currency, like kobo or cents. ₦50 is 5,000 kobo.

**Why it's needed here:** amounts are stored as whole numbers of minor units. Decimals in computers can drift by tiny amounts, which is unacceptable for money.

### Currency
**What it means:** the type of money, like NGN or USD.

**Why it's needed here:** every amount carries its currency explicitly, with no default. The database makes mixing currencies in one transfer impossible.

### Append-only
**What it means:** records can be added but never changed or deleted.

**Why it's needed here:** history must be trustworthy. The database blocks any edit or delete of ledger rows.

### Reversal
**What it means:** a new transfer that exactly undoes an earlier one.

**Why it's needed here:** since history can't be edited, mistakes are fixed by reversing them. Reversals need a special, higher permission than normal transfers.

### Trial balance
**What it means:** a check that all entries across the whole ledger add up to zero.

**Why it's needed here:** it's a quick health check of the entire ledger.

### Reconciliation
**What it means:** re-adding all the entries to check stored balances are correct.

**Why it's needed here:** stored balances are a shortcut. A script re-adds everything to prove they match. After a load test of 17,697 transactions, they all did.

---

## 2. The Database

### PostgreSQL
**What it means:** a popular, powerful, free database.

**Why it's needed here:** it's the single source of truth. It supports the features the safety rules need, like triggers, row locks and delayed checks.

### Database transaction
**What it means:** a group of changes that either all succeed together or are all cancelled.

**Why it's needed here:** a transfer changes several rows: the transfer, two entries, two balances and more. They must never be half-saved.

### Constraint
**What it means:** a rule the database enforces on data, like "balance can't be negative".

**Why it's needed here:** the rules can't be skipped, even by buggy code.

### Trigger
**What it means:** a small piece of code inside the database that runs automatically when data changes.

**Why it's needed here:** triggers block edits and deletes on ledger rows, and check that each transfer's entries add up to zero.

### Deferred check
**What it means:** a check that waits until the end of a database transaction, instead of running after each change.

**Why it's needed here:** entries are added one at a time. After the first, the total isn't zero yet. Waiting until the end means the check sees the whole transfer.

### Foreign key
**What it means:** a rule that a value must match a row in another table, like "this entry's account must exist".

**Why it's needed here:** here, keys combine the account *and* its currency. So an entry in the wrong currency simply can't be saved.

### Balance cache
**What it means:** storing each account's current balance, instead of adding up its entries every time.

**Why it's needed here:** reading a stored balance is 36 times faster. It's updated in the same transaction as the entries, so it's never out of date.

### Migration
**What it means:** a versioned script that changes the database's structure.

**Why it's needed here:** all the tables, rules and triggers are created by a migration, so any database can be set up the same way.

### Index
**What it means:** a lookup structure that makes finding rows fast, like a book's index.

**Why it's needed here:** the right index made account statements 409 times faster. Another index was rejected because it slowed down writes by 14%.

---

## 3. Many Requests at Once

### Concurrency
**What it means:** many requests running at the same time.

**Why it's needed here:** real payment systems get many requests at once. That's when the worst bugs appear.

### Race condition
**What it means:** a bug where the result depends on which request happens to finish first.

**Why it's needed here:** two payments from one account could both see enough money and overdraw it. Locking prevents this.

### Pessimistic locking
**What it means:** locking a row *before* changing it, so others wait their turn.

**Why it's needed here:** it's the default. On a very busy account it handled 145 transfers per second with no conflicts.

### Optimistic locking
**What it means:** not locking, but checking at save time that nobody else changed the row (using a version number). If they did, try again.

**Why it's needed here:** it's offered as an option. On busy accounts, it rejected 46% of attempts, so it's not the default.

### Deadlock
**What it means:** two requests each waiting for a lock the other holds, so both wait forever.

**Why it's needed here:** locking accounts one at a time, always in the same order, makes deadlocks impossible.

### Isolation level
**What it means:** how strictly the database separates transactions running at the same time.

**Why it's needed here:** the project uses the standard level (read committed) plus row locks. Stricter levels would turn clashes into errors, or slow everything down.

### Hot account
**What it means:** an account used by a huge share of transfers, like a platform's fee account.

**Why it's needed here:** everyone queues for its lock. In the load test, the slowest responses took 1,400 ms with a hot account, against 360 ms without.

---

## 4. Safe Retries

### Idempotency
**What it means:** doing the same request twice has the same effect as doing it once.

**Why it's needed here:** apps retry when the connection drops. Without this, a retry could move money twice.

### Idempotency key
**What it means:** a unique ID the client sends with each request.

**Why it's needed here:** the database only lets the first request with that ID through. A repeat gets the original answer back. The same ID with *different* details is rejected.

### Unique constraint
**What it means:** a database rule that no two rows can have the same value in a column.

**Why it's needed here:** it's how the first request with a key "wins". The database itself refuses the second, which is safer than a timed lock that could expire.

### Middleware
**What it means:** code that runs on every request before and after the main logic.

**Why it's needed here:** the duplicate check, logging, security and rate limiting all run as middleware.

---

## 5. The API

### REST API
**What it means:** a common style of API where you send requests to web addresses, like `POST /v1/transfers`.

**Why it's needed here:** it's how apps use the ledger: create accounts, make transfers, reverse them, and read statements.

### API key and scopes
**What it means:** an API key is a secret code identifying a client. Scopes are the permissions attached to it.

**Why it's needed here:** keys are stored only as fingerprints (SHA-256), never in plain form. Reversing money needs its own scope.

### Hash (SHA-256)
**What it means:** a one-way fingerprint of data. You can check a match, but can't turn it back into the original.

**Why it's needed here:** if the database leaked, the real API keys still couldn't be read.

### Rate limiting
**What it means:** capping how many requests a client can make in a time window.

**Why it's needed here:** it protects the system from being overwhelmed. If Redis is down, rate limiting is skipped rather than blocking everyone, because money correctness lives in PostgreSQL.

### Circuit breaker
**What it means:** stopping calls to a failing service for a while, instead of trying again and again.

**Why it's needed here:** if Redis is struggling, the circuit breaker stops the API waiting on it.

### Problem details (RFC 7807)
**What it means:** a standard format for error messages that programs can read.

**Why it's needed here:** every error, even "page not found", comes back in the same format, so apps can handle them consistently.

### Keyset pagination
**What it means:** getting long lists page by page using "continue after this item", instead of "skip the first N items".

**Why it's needed here:** "skip N" gets slower with every page and can count rows twice while new ones arrive. Keyset was about 17 times faster at page depth 5,000.

---

## 6. Sending Events

### Event
**What it means:** a message saying something happened, like "transfer completed".

**Why it's needed here:** other systems, like notifications, need to know about transfers.

### Dual write problem
**What it means:** the risk when saving to two places (a database and a message system) where one can succeed and the other fail.

**Why it's needed here:** announcing a transfer that was never saved, or saving one that's never announced, would both be serious bugs.

### Transactional outbox
**What it means:** saving the event in the same database transaction as the transfer, then sending it afterwards.

**Why it's needed here:** it solves the dual write problem. If the transfer is saved, so is its event. Nothing is lost.

### Relay and SKIP LOCKED
**What it means:** the relay is a background worker that sends saved events. SKIP LOCKED lets several workers each grab different events without waiting on each other.

**Why it's needed here:** events are picked up and sent reliably, even with more than one worker. The trade-off is up to half a second of delay.

### At-least-once delivery
**What it means:** every event is sent at least once, but might occasionally be sent twice.

**Why it's needed here:** it's the price of never losing an event. So consumers must handle duplicates.

### Idempotent consumer
**What it means:** a receiver that ignores events it has already processed.

**Why it's needed here:** it records each event's ID in Redis, so a duplicate is simply skipped.

---

## 7. Proving It Works

### Property-based testing
**What it means:** instead of writing examples by hand, a tool generates many random scenarios and checks a rule always holds.

**Why it's needed here:** it checks that total money is conserved after every random series of operations.

### Load test
**What it means:** sending lots of realistic traffic to see how the system copes.

**Why it's needed here:** it found the limit of about 250 requests per second on a 4-core machine, with no failures.

### Benchmark
**What it means:** a careful speed measurement, run the same way each time.

**Why it's needed here:** every speed claim in the project has a saved benchmark result behind it.

### N+1 query problem
**What it means:** fetching a list with one query, then running one extra query per item, which is very slow.

**Why it's needed here:** fixing it cut 101 queries to 1, making the request about 12 times faster.

### Metrics and structured logs
**What it means:** metrics are live numbers, like requests per second. Structured logs are records written in a consistent, machine-readable format.

**Why it's needed here:** every request gets an ID that appears in every log line, so problems can be traced from start to finish.

### Graceful shutdown
**What it means:** stopping a server cleanly: first refusing new work, then finishing what's in progress.

**Why it's needed here:** a transfer should never be cut off halfway through when the server restarts.
