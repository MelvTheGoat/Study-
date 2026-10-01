# Double-Entry Ledger API: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

## 1. The Problem

Money is the hardest thing to get right in software. Small mistakes can create money from nothing, lose it, or move it twice.

Here are three real dangers. Two payments from the same account at the same moment could both spend the same money and overdraw the account. A phone that loses signal might resend a payment, moving the money twice. And a "payment done" message could be sent out for a payment that never actually saved.

## 2. The Big Idea

This project is the "money engine" behind a payments app. It moves money between accounts, and keeps a permanent, balanced record of every move.

The big idea is to put the safety rules **inside the database itself**, not just in the app's code. Even if the app code had a bug, the database would still refuse to break the rules. Every claim is backed by tests and speed measurements.

## 3. How Double-Entry Works

Double-entry is an old accounting idea. Every transfer is written down twice: once as money *out* of one account, and once as money *in* to another. The two always add up to zero.

For example, to move ₦50 from account A to account B, money is stored in kobo, so ₦50 = 5,000 kobo. A gets −5,000, and B gets +5,000. −5,000 + 5,000 = 0. If the two lines ever didn't add up to zero, money would have appeared or vanished, and the database refuses to save it.

## 4. How a Transfer Works, Step by Step

**Step 1: A request arrives.** An app sends "move ₦50 from A to B". It includes a secret API key (proving who it is) and a unique request ID.

**Step 2: Security checks.** The system checks the API key and its permissions. It also checks the app isn't sending too many requests.

**Step 3: Duplicate check.** The system looks up the request ID. If this exact request was already done, it just returns the saved answer. Money never moves twice.

**Step 4: Check the details strictly.** The amount must be a whole number, the currency must be stated, and both accounts must exist and use the same currency.

**Step 5: Lock the accounts.** Both accounts are locked, always in the same order, so only this transfer can change them right now. It's like a "do not disturb" sign.

**Step 6: Check there's enough money.** Then it writes the transfer and its two entries (−5,000 and +5,000).

**Step 7: Update the balances.** Each account's stored balance is updated in the same step.

**Step 8: Write the message and the record.** A "transfer done" message is saved in a special outbox table, along with an audit record. All of this happens in one database transaction.

**Step 9: Save everything, or nothing.** When saving, the database checks the entries add up to zero. If anything is wrong, every change is cancelled. Nothing is ever half-saved.

**Step 10: Send the message later.** A background worker picks up messages from the outbox and sends them through Redis Streams (a fast message pipe) to other systems.

## 5. The Clever Parts

**Rules the database enforces itself.** Entries must add up to zero, and history can't be edited or deleted. Balances can't go negative (unless an account is allowed to), and currencies can't be mixed in one transfer. Tests even try to break these rules directly in the database, to prove it refuses.

**Mistakes are fixed by reversing, not editing.** Because history can't be changed, a wrong transfer is fixed with a new transfer that undoes it. Reversing money needs a special, higher permission.

**Locking in a fixed order.** Two requests could each lock one account and wait forever for the other. That's called a deadlock. Always locking in the same order makes it impossible.

**Safe retries.** The first request with a given ID "wins" thanks to a database rule that no two can share the same ID. A repeat gets the original answer. The same ID with *different* details is rejected.

**The outbox.** Sending "transfer done" *before* saving could announce a transfer that never happened. Saving the message together with the transfer, then sending it afterwards, means messages are never lost and never false.

**Fast balance reads.** Each account stores its current balance, which is 36 times faster to read than adding up every entry. A checking script re-adds everything to prove the stored balances are right.

**Redis is helpful, never essential.** Redis handles speed limits and messages. If Redis fails, the speed limit is simply skipped. Money stays correct, because it lives in PostgreSQL.

## 6. The Important Words

- **Ledger**: the permanent record of every money movement.
- **Double-entry**: every transfer written as money out and money in, adding up to zero.
- **Entry**: one line in the ledger: an account and an amount.
- **Minor units**: the smallest unit of a currency, like kobo or cents.
- **API**: a way for other programs to ask this system to do things.
- **Database transaction**: a group of changes that all succeed or all fail together.
- **Lock**: a "do not disturb" sign on a row of data.
- **Idempotency**: sending the same request twice has the same effect as once.
- **Append-only**: records can be added, but never changed or deleted.
- **Outbox**: saving outgoing messages with the transfer, then sending them later.

## 7. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **FastAPI**: the web service that receives requests.
- **PostgreSQL**: the database, and the single source of truth.
- **SQLAlchemy and asyncpg**: let Python talk to PostgreSQL efficiently.
- **Alembic**: sets up the database's tables and rules.
- **Redis**: speed limits, the message pipe, and duplicate protection for messages.
- **Prometheus client**: shares live numbers about how the system is doing.
- **pytest and Hypothesis**: run tests, including thousands of random scenarios.
- **Locust**: simulates many users at once to test the load.
- **Docker Compose**: starts everything with one command.

## 8. How Good Is It?

- **Tests**: 140 automatic tests pass, against a real PostgreSQL and Redis, not pretend ones.
- **Busy accounts**: with locking, a very busy account handled about 145 transfers per second with no clashes. The alternative method managed 41, and rejected 46% of attempts.
- **Load test**: the system handled about 250 requests per second on a 4-core machine, with 0 failures. Every balance checked out afterwards.
- **Speed fixes**: the right index made account statements 409 times faster. Fixing a repeated-query problem turned 101 database queries into 1.

## 9. What's Weak or Missing

- Everything runs on a single PostgreSQL database. Growing much bigger would need splitting data across several.
- Its limit beyond about 250 requests per second on one shared machine isn't known.
- It doesn't support exchanging between currencies yet.
- A reversal must undo a whole transfer, not part of one.
- Messages can take up to half a second to be sent.
- The audit record grows forever, with no clean-up rule yet.

## 10. What This Project Shows You Can Do

- Build money software that stays correct under pressure.
- Use the database itself to enforce important rules.
- Handle many requests at once without overdrafts or deadlocks.
- Make every request safe to retry.
- Back every claim with tests and measurements.

## 11. Ten Things to Remember

1. It's the money engine behind a payments app.
2. Every transfer is two entries that add up to zero.
3. Money is stored as whole numbers of kobo or cents, never decimals.
4. The key rules live inside the database, so nothing can skip them.
5. History is never edited, and mistakes are fixed with reversals.
6. Accounts are locked in a fixed order to stop overdrafts and deadlocks.
7. A unique request ID makes retries safe.
8. The outbox makes sure "transfer done" messages are never lost or false.
9. Redis helps with speed, but money never depends on it.
10. It handled about 250 requests per second, with 0 failures.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
