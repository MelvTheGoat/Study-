# Stocks (Stock Research)

Repo: https://github.com/MelvTheGoat/Stocks

## In short

A private website for understanding one company at a time, American or Nigerian. It shows the numbers, explains each one as you look at it, and runs eight warning checks, but it never says buy or sell. US accounts come from the SEC, US prices from Twelve Data's free plan, and Nigerian figures from the companies' own results PDFs (prices typed in by hand, because the Nigerian Exchange forbids automated collection). It rebuilds itself every weekday night on GitHub Actions and is published on Cloudflare Pages behind a login.

The repo began (18 Sep 2026) as an "eval-first" AI agent for stock questions. On 6 Oct 2026 the agent, eval harness and Kaggle runner were dropped and it became this research tool. These reports describe the current version.

## Key facts

| | |
|---|---|
| Language | Python 3.11+ |
| Core tech | httpx, DuckDB + Parquet, Jinja2 (plain HTML site), pypdf, PyYAML |
| Data | SEC company facts (US accounts), Twelve Data free plan (US prices), company PDFs + hand-typed prices (Nigeria) |
| On each company page | 29 figures, each with a source and date (or a reason it's missing), 8 warning checks, 5-year P/E history, peer comparison, latest-report changes |
| Tests | 436 pass, 1 skipped (my run). CI also runs them with the network off. |
| Cost | Free: SEC, Twelve Data free plan, GitHub Actions, Cloudflare Pages + Access, ntfy |
| Not yet | Any Nigerian company entered. Whether the site is live isn't shown in the repo. |

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
