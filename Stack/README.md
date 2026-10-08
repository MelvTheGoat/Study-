# Stack ("Reckon")

Repo: https://github.com/MelvTheGoat/Stack

## In short

Reckon matches incoming payments (card, dedicated account, bank transfer, cash) to invoices. Exact rules go first, then an 18-feature calibrated logistic model with a cost-derived threshold. Everything uncertain goes to a review queue ranked by money at risk, with an append-only audit log. Built for Paystack in Nigeria, with integer-kobo money.

## Key facts

| | |
|---|---|
| Language | Python 3.11 (mypy strict) |
| Core tech | FastAPI, SQLAlchemy, Pydantic, httpx, Jinja2, standard-library ML |
| Results (synthetic month, reproduced) | 76.1% auto-closed, 0 wrong, recall 80.1%, first 40 reviews = 91% of the money at risk |
| Threshold | 0.85, from a ₦60 vs ₦5,000 cost ratio |
| Tests | 494 pass with the optional LLM extra. 8 fail without it. |
| Deployed | Not yet |
| In progress (Oct 2026) | Using a business's own books, on the `feat/own-books` branch. Main still runs on practice data. |

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
