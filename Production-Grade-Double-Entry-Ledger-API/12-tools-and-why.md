# Double-Entry Ledger API: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like idempotency or the outbox, are explained in [11-technical-terms.md](11-technical-terms.md).

The main idea: **PostgreSQL is the source of truth**. Everything that decides whether money is correct lives there. Redis helps with speed and messaging, but money never depends on it.

---

## The Language and API

### Python 3.12
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** the whole API is written in it, with every value's kind written down and checked.

### FastAPI
**What it is:** a Python tool for building APIs (ways for programs to ask for things). It can handle many requests at once while waiting on the database.

**Why it's used here:** it serves every endpoint: accounts, transfers, reversals, statements and the trial balance. Its middleware hooks run the security, duplicate check and logging on every request.

---

## The Database

### PostgreSQL 16
**What it is:** a popular, powerful, free database.

**Why it's used here:** it's the source of truth. It enforces the money rules itself, using triggers, delayed checks, row locks and constraints. Its "skip locked" feature lets several workers share the event queue safely.

### SQLAlchemy (async) and asyncpg
**What it is:** SQLAlchemy is a Python tool for working with databases from code. asyncpg is a fast driver that lets Python talk to PostgreSQL without blocking while it waits.

**Why it's used here:** together they run every database query, and let the API keep serving other requests while waiting for answers.

### Alembic
**What it is:** a tool for "migrations": versioned scripts that set up or change the database's structure.

**Why it's used here:** one migration creates all the tables, rules, triggers and indexes. Any new database can be set up identically.

---

## Speed and Messaging

### Redis 7
**What it is:** a very fast, in-memory data store, often used for counters, caches and message queues.

**Why it's used here:** it does three jobs:

1. **Rate limiting:** counts each client's recent requests, in one step using a small Lua script (Lua is a tiny programming language Redis can run).
2. **Redis Streams:** a message pipe that carries "transfer done" events to other systems.
3. **Duplicate protection for consumers:** remembers which events were already processed.

If Redis fails, rate limiting is skipped rather than blocking everyone. Money stays correct because it lives in PostgreSQL.

---

## Watching It Run

### Prometheus client
**What it is:** a tool that exposes live numbers (metrics), like requests per second and error counts, for Prometheus, a popular monitoring system.

**Why it's used here:** it shows how the API is doing in real time.

### Structured JSON logs
**What it is:** log lines written as JSON (simple labelled data) instead of free text.

**Why it's used here:** every request gets an ID that appears in all its log lines, so one request can be traced from start to finish. This uses Python's own tools, not an extra install.

---

## Testing and Proving

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code works.

**Why it's used here:** 140 tests pass against a *real* PostgreSQL and Redis, not fake ones. Some try to break the rules directly in the database, to prove it refuses.

### Hypothesis
**What it is:** a tool for property-based testing: it invents many random scenarios and checks a rule always holds.

**Why it's used here:** it checks that total money is always conserved, however operations are mixed.

### Locust
**What it is:** a tool for load testing, which simulates many users sending requests at once.

**Why it's used here:** it found the limit of about 250 requests per second on a 4-core machine, with no failures. Every balance still checked out afterwards.

### Benchmark scripts
**What it is:** the project's own scripts that carefully measure speed, like comparing locking styles or index choices.

**Why it's used here:** every performance claim is backed by a saved result in the project.

---

## Running It

### Docker Compose
**What it is:** Docker packs an app into a box, called a container, that runs the same anywhere. Compose starts several boxes together from one settings file.

**Why it's used here:** one command starts the API, PostgreSQL, Redis, the event worker and the database setup.

### GitHub Actions
**What it is:** a free service from GitHub that runs checks automatically when code changes.

**Why it's used here:** it starts real PostgreSQL and Redis containers, then runs the full test suite against them for every change.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.12 | The language, fully typed |
| FastAPI | The API and its middleware |
| PostgreSQL 16 | The source of truth, enforcing the money rules |
| SQLAlchemy + asyncpg | Talk to PostgreSQL without blocking |
| Alembic | Sets up the database structure |
| Redis 7 | Rate limits, event pipe, duplicate protection |
| Prometheus client | Live metrics |
| JSON logs | Traceable requests |
| pytest + Hypothesis | Tests, including random scenarios |
| Locust | Load testing |
| Docker Compose | Start everything with one command |
| GitHub Actions | Tests every change against real services |
