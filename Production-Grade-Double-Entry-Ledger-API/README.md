# Production-Grade-Double-Entry-Ledger-API

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

## In short

A double-entry ledger and payments API (Python 3.12, FastAPI, PostgreSQL 16, Redis 7) where correctness is enforced by the database and proven by tests and benchmarks: balanced entries via a deferred trigger, an append-only ledger, ordered row locking against overdraft races, unique-constraint idempotency, and a transactional outbox for events.

## Key facts

| | |
|---|---|
| Language | Python 3.12 |
| Core tech | FastAPI, SQLAlchemy async/asyncpg, PostgreSQL 16, Redis 7, Alembic, locust |
| Locking benchmark | Pessimistic 144.8 tps vs optimistic 41.3 tps (46% conflicts) on a hot account |
| Load test | ~250–260 req/s ceiling on a shared 4-vCPU box. 0 failures. Reconciliation exact. |
| Tests (my run) | 140 passing against real Postgres 16 + Redis |

## Files

| File | What's in it |
|---|---|
| [00-start-here.md](00-start-here.md) | **Start here.** The whole project in simple English (good for NotebookLM) |
| [01-system-design.md](01-system-design.md) | Parts, diagram, stack, data flow, trade-offs, 10x |
| [02-how-to-write-the-system-design.md](02-how-to-write-the-system-design.md) | Whiteboard steps |
| [03-linkedin-post.md](03-linkedin-post.md) | LinkedIn post |
| [04-blog-post.md](04-blog-post.md) | Blog post |
| [05-explain-to-non-technical.md](05-explain-to-non-technical.md) | Plain-language explanation |
| [06-explain-to-technical.md](06-explain-to-technical.md) | Engineer-level explanation |
| [07-defend-in-interview.md](07-defend-in-interview.md) | Pitch, questions, weak spots |
| [08-ten-points-to-know.md](08-ten-points-to-know.md) | 10 key facts |
| [09-what-this-proves-i-know.md](09-what-this-proves-i-know.md) | Skills and related topics |
| [10-system-design-for-beginners.md](10-system-design-for-beginners.md) | The whole system explained step by step, for beginners |
| [11-technical-terms.md](11-technical-terms.md) | Every technical term: what it means and why it's used here |
| [12-tools-and-why.md](12-tools-and-why.md) | Every tool: what it is and why it was picked |
| [13-talking-it-through.md](13-talking-it-through.md) | The whole build talked through like a conversation, naming every file as it is created |
