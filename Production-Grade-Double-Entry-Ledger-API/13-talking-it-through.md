# Double-Entry Ledger API: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. Money is the hardest thing to get right in software. Small mistakes can create money from nothing, lose it, or move it twice.

Here are three real dangers. Two payments from the same account at the same moment could both spend the same money. A phone that loses signal resends a payment, and the money moves twice. And a "payment done" message goes out for a payment that never actually saved.

So here's our project: the money engine behind a payments app. It moves money between accounts, and keeps a permanent, balanced record of every move. And the big idea is to put the safety rules **inside the database itself**. Even if the app code had a bug, the database would still refuse to break the rules.

## So what do we need?

1. **Settings**, and a database connection that does one job per transaction.
2. **The tables**, designed so bad data can't even be stored.
3. **Database rules** that check every transfer balances, and that history can't be edited.
4. **The ledger core**: post a transfer, read a balance, reverse a mistake.
5. **Locking**, so two payments at once can't overdraw an account.
6. **The API**: accounts, transfers, reversals, statements.
7. **Security, rate limits, and safe retries.**
8. **Messages to other systems** that are never lost or false.
9. **Logging, metrics, and a calm way to fail and shut down.**
10. **Proof**: tests against a real database, a consistency check, and speed measurements.

## Step zero: set up the workshop

First, a `README.md`, the front page, and a `.gitignore`. `pyproject.toml` describes the project and its tool settings.

Two install lists: `requirements.txt` has only what the live service needs, and `requirements-dev.txt` adds testing and speed-testing tools. As the notes say, the extras never reach the live box. `.env.example` is a template for settings.

The code lives in `src/`. Its `__init__.py` says simply: "A double-entry ledger and payments API." `src/config.py` reads every setting from the environment, especially anything that affects correctness or capacity, like connection pool sizes.

And `src/db.py` enforces a rule: **one database transaction per piece of business work**. A transfer writes several rows, and they must all save together or not at all.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `pyproject.toml`: project details and tool settings.
> - `requirements.txt` and `requirements-dev.txt`: live needs, and testing extras.
> - `.env.example`: a template for settings.
> - `src/__init__.py`: marks the code folder as a package.
> - `src/config.py`: every setting, read from the environment.
> - `src/db.py`: one database transaction per piece of work.

## Step one: tables that can't hold bad data

Now the tables, in `src/ledger/models.py`. Its notes tell you to read it together with `ERD.md`, a document describing the 7 tables. And the idea behind both is powerful: every rule exists to make a specific kind of corruption *impossible to store*.

For example, every amount is a whole number of the smallest unit, like kobo or cents, with its currency always stated. A balance can't go negative unless the account allows it. And an entry's currency must match its account's currency, enforced by the database, so mixing currencies simply can't happen.

`src/ledger/enums.py` holds the fixed lists of words, like account types and statuses. They're kept in their own file so other parts can use them without loading the database machinery. `src/ledger/__init__.py` marks the folder as a package.

> **📁 Files we just created**
> - `src/ledger/__init__.py`: marks the ledger folder as a package.
> - `src/ledger/models.py`: the tables, with rules that block corruption.
> - `src/ledger/enums.py`: fixed lists of words, like statuses.
> - `ERD.md`: the 7 tables, and what each rule prevents.

## Step two: rules the database enforces itself

Some rules can't be written as a simple column rule. So we use Alembic, a tool for "migrations", which are versioned scripts that set up the database.

`alembic.ini` is Alembic's settings. `migrations/env.py` connects it to the database. Migrations run one step at a time, so they don't need the fast, everything-at-once style the app uses. `migrations/script.py.mako` is the template for new migrations.

And the important one: `migrations/versions/0001_initial_schema.py`. It creates every table, then installs the special rules. A delayed check makes sure every transfer's entries add up to exactly zero, and that the number of entries matches what was declared. It waits until the end of the transaction, because entries are added one at a time.

It also installs rules that **block any edit or delete** of ledger history. Mistakes are fixed with a reversal, never by changing the past.

Tested in `tests/test_ledger_invariants.py`. And here's what's clever: several of these tests deliberately go *around* the app and write bad data straight into the database, to prove the database itself refuses it.

> **📁 Files we just created**
> - `alembic.ini`: Alembic's settings.
> - `migrations/env.py`: connects Alembic to the database.
> - `migrations/script.py.mako`: the template for new migrations.
> - `migrations/versions/0001_initial_schema.py`: creates the tables and the balance and no-editing rules.
> - `tests/test_ledger_invariants.py`: proves the database itself refuses bad data.

## Step three: the ledger core, and locking

`src/ledger/service.py` is the core. Everything that changes money goes through **one** function:

```python
async def post_transaction(
```

That's the only door. It writes the transfer, its entries, and updates each account's stored balance in the same step. It also handles reading balances and statements, and reversing transfers.

`src/ledger/errors.py` defines the problems the ledger can report, like "not enough money". Each one maps to exactly one kind of error message in the API.

Then `src/ledger/locking.py`, and its notes call it "the heart of the project". Here's the problem it solves. Account A has 100.

Two transfers of 60 arrive at the same moment. Both read 100, both think "fine", and the account ends up overdrawn.

So before changing an account, we lock it. Others have to wait their turn. And we always lock accounts in the same order, smallest ID first, one at a time.

That stops two transfers each holding one lock and waiting forever for the other, which is called a deadlock. There's also an optional "optimistic" style, which uses a version number instead of waiting.

Tested in `tests/test_concurrency.py`, and its notes say it's "the most important file in the repository". It fires genuinely simultaneous transactions, on separate database connections, to make the races actually happen.

Let's pause. We have tables that block corruption, database rules that check every transfer balances and block edits, one door for changing money, and locking that stops overdrafts. That's the safe core. Now the API.

> **📁 Files we just created**
> - `src/ledger/service.py`: the one door for changing money, plus balances and reversals.
> - `src/ledger/errors.py`: the problems the ledger can report.
> - `src/ledger/locking.py`: locking that stops overdrafts and deadlocks.
> - `tests/test_concurrency.py`: fires real simultaneous transfers to test the races.

## Step four: the API

Now the API, built with FastAPI. `src/api/app.py` puts the application together and handles starting up and shutting down. `src/api/__init__.py` marks the folder as a package.

`src/api/schemas.py` defines what goes in and out. And its notes are firm: money is a whole number plus a currency, everywhere, both ways. No decimals at all.

The routes, in `src/api/routes/`:

- `accounts.py`: create, list and read accounts, plus balances and statements.
- `transfers.py`: transfers, transactions and reversals.
- `admin.py`: admin jobs, needing the admin permission.
- `ops.py`: health checks, metrics and the trial balance.

And `src/api/routes/__init__.py` mounts them all under `/v1`, except the operations ones.

There's a smart point in `ops.py`. "Is it alive?" and "is it ready?" are different questions. Mixing them up can cause outages, so they're separate checks.

`src/api/errors.py` gives every error the same standard shape, even ones the web framework would normally show differently. `src/api/pagination.py` uses "continue after this item" paging instead of "skip the first N". Skipping can count rows twice while new ones arrive, and it gets slower the deeper you go.

> **📁 Files we just created**
> - `src/api/__init__.py`: marks the API folder as a package.
> - `src/api/app.py`: puts the application together.
> - `src/api/schemas.py`: what goes in and out, with money as whole numbers.
> - `src/api/routes/__init__.py`: mounts the routes under `/v1`.
> - `src/api/routes/accounts.py`, `transfers.py`, `admin.py`, `ops.py`: the endpoints.
> - `src/api/errors.py`: one standard shape for every error.
> - `src/api/pagination.py`: safe paging for long lists.

## Step five: security, limits and safe retries

**Security.** `src/api/auth.py` checks API keys. Keys look like `lk_<env>_<body>`, and only a fingerprint (a SHA-256 hash) is stored, never the key itself. Each key has permissions, and reversing money needs its own, higher one.

**Rate limits.** `src/api/ratelimit.py` caps how many requests each key can make, using Redis, a very fast in-memory store. And if Redis is down, the limit is simply skipped rather than blocking everyone. Money correctness lives in PostgreSQL, not Redis.

**Safe retries.** `src/api/idempotency.py`. Apps resend requests when the connection drops. So each request carries a unique ID, and the database only lets the first one through:

```python
ON CONFLICT (api_key_id, endpoint, idempotency_key) DO NOTHING
```

That means "if this ID already exists, do nothing". A repeat gets the original answer back. The same ID with *different* details is rejected. And if the first one is still running, the repeat is told to wait and try again.

`src/api/middleware.py` runs on every request: it tags it with an ID, checks the key, applies the rate limit, and tracks requests for a clean shutdown. `src/api/deps.py` lets each endpoint read who's calling, which the middleware already worked out.

Tests: `tests/test_idempotency.py`, where the key test is two identical requests arriving at the same moment. And `tests/test_api.py` covers keys, permissions, errors, paging and rate limits.

> **📁 Files we just created**
> - `src/api/auth.py`: API keys stored as fingerprints, with permissions.
> - `src/api/ratelimit.py`: per-key limits in Redis, skipped if Redis is down.
> - `src/api/idempotency.py`: makes every request safe to resend.
> - `src/api/middleware.py`: request ID, key check, rate limit and shutdown tracking.
> - `src/api/deps.py`: lets endpoints read who's calling.
> - `tests/test_idempotency.py` and `tests/test_api.py`: retry and API tests.

## Step six: messages that are never lost or false

Other systems need to know when a transfer happens. But sending the message straight from the request is risky. If you send it and then the save fails, you've announced a transfer that never happened. If you save and then the send fails, it's lost.

So `src/events/outbox.py` saves the message in the **same database transaction** as the transfer. If the transfer is saved, so is its message. If not, neither is.

Then `src/events/relay.py` sends the saved messages later, through Redis Streams, a fast message pipe. It uses `SKIP LOCKED`, which lets several relays share the work without grabbing the same message.

Messages might occasionally arrive twice, but never get lost. So `src/events/consumer.py` is an example receiver that ignores messages it's already handled.

`src/worker.py` runs the relay, a clean-up of old retry records, and a metrics refresh, as a separate program from the API. That way a slow relay can't slow down the API. `src/events/__init__.py` marks the folder as a package.

Tested in `tests/test_outbox.py`. Its point is that the message and the money share a fate: if the transfer is cancelled, so is its message.

> **📁 Files we just created**
> - `src/events/__init__.py`: marks the events folder as a package.
> - `src/events/outbox.py`: saves messages together with the transfer.
> - `src/events/relay.py`: sends saved messages, safely shared between workers.
> - `src/events/consumer.py`: an example receiver that ignores repeats.
> - `src/worker.py`: runs the relay and clean-ups as a separate program.
> - `tests/test_outbox.py`: checks the message and the money share a fate.

Let's circle back. We have the safe core, a full API with security, limits and safe retries, and messages that are never lost or false. Now, how do we watch it, and how does it fail safely?

## Step seven: watching it, and failing calmly

`src/observability/logging.py` writes one line per event, as JSON, with the request ID on every line. `src/observability/context.py` carries that request ID through the code automatically, so nothing has to pass it along by hand. And `src/observability/metrics.py` has a deliberately small set of live numbers, only the ones that answer "is it healthy?" and "why is it slow?". `src/observability/__init__.py` marks the folder as a package.

Then failing safely. `src/resilience/__init__.py` states the policy in one place: what crashes, what degrades, and what refuses. `src/resilience/circuit.py` is a circuit breaker.

When Redis is down, it stops calling it for a while, so requests don't all wait for timeouts. And `src/resilience/shutdown.py` handles a clean shutdown: stop taking new work, finish what's in flight, then exit.

Tested in `tests/test_resilience.py`.

> **📁 Files we just created**
> - `src/observability/__init__.py`, `logging.py`, `context.py`, `metrics.py`: logs, request IDs and live numbers.
> - `src/resilience/__init__.py`: the failure policy, stated once.
> - `src/resilience/circuit.py`: stops calling a service that's down.
> - `src/resilience/shutdown.py`: finishes in-flight work before stopping.
> - `tests/test_resilience.py`: tests the failure handling.

## Step eight: proving it

`scripts/reconcile.py` proves the ledger is consistent. It re-adds every entry and checks the stored balances match, and repairs them if not. Its notes say this is what makes the stored balance a defensible design. Tested in `tests/test_reconciliation.py`.

`tests/test_property_invariants.py` uses property-based testing. Instead of examples someone thought of, it generates random sequences of operations and checks that money is always conserved. `tests/conftest.py` sets up every test against a **real PostgreSQL**, not a pretend one, because the whole point is that correctness lives in the database.

Then the speed proof. `scripts/seed.py` fills a database with over 100,000 accounts and over a million entries, deliberately uneven, because perfectly even fake data hides real problems. `scripts/benchmark.py` measures queries.

`scripts/locking_bench.py` compares the two locking styles under pressure. `scripts/index_write_cost.py` measures what each index costs when *writing*, not just reading. `scripts/__init__.py` marks the folder as a package.

## The load test

`loadtest/locustfile.py` is a load test, using Locust to simulate many users at once. `loadtest/prepare.py` creates a key and funded accounts for it first. `loadtest/__init__.py` marks that folder as a package.

The results are saved in `bench-results/`: `locking.json`, `queries.json`, and the load test results `loadtest-20u_stats.csv`, `loadtest-50u_stats.csv` and `loadtest-50u-nohot_stats.csv`. That last one is 50 users without a single very busy account. A `.gitkeep` keeps the folder.

> **📁 Files we just created**
> - `scripts/reconcile.py`: re-adds every entry to prove balances are right.
> - `tests/test_reconciliation.py`: tests the consistency check.
> - `tests/test_property_invariants.py`: random scenarios that check money is conserved.
> - `tests/conftest.py`: runs every test against a real PostgreSQL.
> - `scripts/seed.py`: fills a database with realistic, uneven data.
> - `scripts/benchmark.py`, `scripts/locking_bench.py`, `scripts/index_write_cost.py`: speed measurements.
> - `scripts/__init__.py` and `loadtest/__init__.py`: mark those folders as packages.
> - `loadtest/locustfile.py` and `loadtest/prepare.py`: the load test and its setup.
> - `bench-results/locking.json`, `queries.json`, `loadtest-20u_stats.csv`, `loadtest-50u_stats.csv`, `loadtest-50u-nohot_stats.csv`, `.gitkeep`: the saved results.

## Step nine: packaging and running

`Dockerfile` packs it into a box called a container, and `.dockerignore` says what to leave out. `docker/entrypoint.sh` lets the same box play three roles: the API, the worker, or a one-time database setup. Why run the setup separately? Because if every copy of the API set up the database at start-up, they'd race each other.

`docker-compose.yml` starts the whole local stack: the API, PostgreSQL, Redis, the worker, and the one-time setup. Everything is local and free. And `.github/workflows/ci.yml` runs the full tests on every change, against real PostgreSQL and Redis.

> **📁 Files we just created**
> - `Dockerfile` and `.dockerignore`: pack it into a box.
> - `docker/entrypoint.sh`: one box, three roles.
> - `docker-compose.yml`: starts the whole local stack.
> - `.github/workflows/ci.yml`: tests every change against real services.

## Step ten: writing it all down

- `ARCHITECTURE.md`: the shape of the system, with a diagram.
- `DECISIONS.md`: 16 recorded decisions and their reasons, like "lock rows by default" and "use an outbox".
- `RUNBOOK.md`: what to do when something goes wrong, written for someone who's just been woken up.
- `CODE_WALKTHROUGH.md`: the project's own build-order walkthrough.
- `CODE_WALKTHROUGH.pdf`: the same walkthrough as a PDF.
- `scripts/md_to_pdf.py`: turns documents into phone-friendly PDFs.

> **📁 Files we just created**
> - `ARCHITECTURE.md`: the system's shape.
> - `DECISIONS.md`: the 16 recorded decisions.
> - `RUNBOOK.md`: what to do when things go wrong.
> - `CODE_WALKTHROUGH.md` and `CODE_WALKTHROUGH.pdf`: the project's own walkthrough.
> - `scripts/md_to_pdf.py`: turns documents into PDFs.

## So, how's it doing?

- **140 tests pass**, against a real PostgreSQL and Redis.
- **Locking**: on a very busy account, locking handled about 145 transfers a second with no clashes. The optimistic style managed 41, and rejected 46% of attempts.
- **Load test**: about 250 requests a second on a 4-core machine, with 0 failures. Every balance checked out afterwards.
- **Speed fixes**: the right index made statements 409 times faster.

## What's still missing?

- **One PostgreSQL database** does everything. Growing much bigger would need splitting the data.
- **The limit beyond about 250 requests a second** on one shared machine isn't known.
- **No currency exchange** yet.
- **Reversals must undo a whole transfer.**
- **Messages can take up to half a second** to be sent.
- **The audit record grows forever.**

## Let's put it all together

So let's look at it in one breath.

We set up **one database transaction per piece of work**. We designed **tables that can't hold bad data**, and **database rules** that check every transfer balances and block any edit of history. We built **one door for changing money**, and **locking in a fixed order** to stop overdrafts and deadlocks.

Then the **API**, with money as whole numbers everywhere, **security** with fingerprinted keys, **rate limits** that never block money, and **safe retries** using a unique ID. We made messages **never lost or false** with an outbox, added **logging and metrics**, and made it **fail calmly**.

Finally, we **proved it**: tests against a real database, random scenario tests, a consistency check, speed measurements and a load test.

Notice how it links. Putting the rules in the database in step two is why the tests in step eight can write bad data directly and watch it fail. The one-door rule in step three is what lets the outbox in step six save every message alongside its transfer. And locking in a fixed order is what makes the busy-account numbers so strong.

That's the project. Money that can't be lost, created or moved twice, with proof.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
