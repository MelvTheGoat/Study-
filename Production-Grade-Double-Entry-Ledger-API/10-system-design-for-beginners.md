# Double-Entry Ledger API: System Design for Beginners

This project is the "money engine" behind a payments app. It moves money between accounts and keeps a permanent, balanced record of every move. It's built so money can never be created or lost by accident, even when many requests arrive at once, requests are retried, or parts of the system fail. The safety rules live inside the database itself, and tests and speed measurements back up every claim.

## Key Terms

- **Ledger**: the permanent record of every money movement, like a bank's master book.
- **Double-entry**: every transfer is written twice, once as money out and once as money in, so the two always add up to zero.
- **API**: a way for other programs to ask this system to do things, like "move ₦50 from A to B".
- **Database**: an organised store of data in tables. Here it's PostgreSQL, a popular free one.
- **Transaction (database)**: a group of changes that either all happen or none happen.
- **Idempotency**: sending the same request twice has the same effect as sending it once.
- **Lock**: a "do not disturb" sign on a row of data, so only one request changes it at a time.
- **Event**: a message saying something happened, like "transfer completed", for other systems to react to.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Money must never be created, lost or moved twice. Speed matters, but correctness matters more.

**Step 2: Figure out the data.** We need accounts, transfers, and the entries (the money-out and money-in lines) for each transfer. Balances can be worked out by adding up entries.

**Step 3: Sketch the main parts.** Requests pass security checks and a duplicate check, then the ledger service writes balanced entries. Events are sent out afterwards.

**Step 4: Walk through one transfer.** Follow "move ₦50 from A to B" from the request to the saved record.

**Step 5: Decide how to know it works.** Test the rules under pressure: many requests at once, retries, crashes. Measure the speed too.

**Step 6: Plan for problems.** Two payments at once could overdraw an account, a retry could pay twice, and a message could announce a transfer that never happened. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Store every amount as a **whole number** of the smallest unit, like kobo or cents.
- Make every transfer's entries add up to exactly **0**.
- **Never** edit or delete history. Mistakes are fixed with a reverse transfer.
- Handle about **250** requests per second on a 4-core test machine.

What does one transfer look like? Step by step, for ₦50:

1. ₦50 = 50 × 100 = **5,000 kobo**.
2. Entry 1: account A gets **−5,000**. Entry 2: account B gets **+5,000**.
3. −5,000 + 5,000 = **0**, so the transfer balances.
4. At 250 requests a second, that's 250 × 60 = **15,000** requests a minute.

### The Big Picture (Step 3)

```
   [Client app]
        |
        v
   [Security: API key + rate limit]
        |
        v
   [Duplicate check (idempotency)]
        |
        v
   [Ledger Service: lock accounts, write entries]
        |
        v
   [PostgreSQL: the source of truth]
        |
        v
   [Outbox Relay] --> [Redis Streams] --> [Other systems]
```

Step by step:

1. A client app sends "move ₦50 from A to B", with an API key and a unique request ID.
2. Security checks the key and that this client isn't sending too many requests.
3. The Duplicate Check looks up the request ID. If it's already done, it returns the saved answer instead of paying again.
4. The Ledger Service locks both accounts, in a fixed order, and checks A has enough money.
5. It writes the transfer, its two entries, the new balances and an "event" row, all in one database transaction.
6. On saving, the database itself checks the entries add up to 0. If not, everything is cancelled.
7. Later, the Outbox Relay sends the event through Redis Streams (a fast message pipe) to other systems.

### The Main Parts (Step 3)

**Rules Inside the Database.** The key rules live in PostgreSQL, not just in the app code. Entries must sum to 0, history can't be edited, balances can't go negative (unless an account allows it), and currencies can't be mixed. It's like a vault that refuses bad deposits by itself, even if a clerk makes a mistake.

**Locking.** Two payments from the same account at the same moment could both see enough money and overdraw it. So each account is locked while it's being changed, always in the same order to avoid two requests waiting on each other forever. It's like a single-person changing room with a "busy" sign.

**Duplicate Check.** Phones lose signal and apps retry. Each request carries a unique ID, and the database only lets the first one through. A repeat gets the original answer back, like a shop recognising a receipt that's already been refunded.

**Balance Cache.** Each account stores its current balance, updated in the same step as the entries. Reading it is 36 times faster than adding up every entry. A checking script re-adds everything to prove the stored balance is right.

**Outbox.** Sending "transfer done" to other systems *before* saving could announce a transfer that never happened. So the message is saved in the same transaction as the transfer, then sent afterwards. It's like writing a letter into the same diary entry as the event, and posting it later.

Using a version number instead of locks, called optimistic locking, is also supported (advanced - skip for now).

### How We Know It's Working (Step 5)

- **Tests**: 140 automatic tests pass against a real PostgreSQL and Redis. Some try to break the rules directly in the database and check it refuses.
- **Locking speed**: on a very busy account, locking handled **145** transfers per second with no conflicts. The alternative managed 41 and rejected 46% of attempts.
- **Load test**: about **250** requests per second, with **0** failures, and every balance checked out afterwards.
- **Fast reads**: the right index made a statement query **409 times** faster.

### What Can Go Wrong (Step 6)

- **Two payments at once.** Locking plus the "no negative balance" rule stops overdrafts.
- **A request is retried.** The duplicate check makes sure money only moves once.
- **A crash mid-transfer.** The database transaction means either everything is saved or nothing is.
- **One database is the limit.** Everything runs on a single PostgreSQL server. Growing further would need splitting data across servers.

## Quick Recap

- Every transfer is two entries that add up to zero.
- Store money as whole numbers of the smallest unit, never decimals.
- Put the key rules inside the database, where nothing can skip them.
- Lock accounts in a fixed order, and make every request safe to repeat.
- Save outgoing messages with the transfer, then send them afterwards.
