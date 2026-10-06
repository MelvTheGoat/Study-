# Stocks

Repo: https://github.com/MelvTheGoat/Stocks

## In short

An agent that will answer factual questions about US stocks and Nigerian (NGX) stocks, built evaluation-first: the test that measures it comes before the agent. **Early stage:** the run config, the model client (cache, retries, logging), the tests and CI are built. The eval set, the agent and the data pipeline are not built yet.

## Key facts

| | |
|---|---|
| Language | Python 3.10+ |
| Core tech | Pydantic, httpx, PyYAML. Planned: vLLM, Qwen2.5-7B-Instruct-AWQ, Kaggle T4 |
| Tests | 64 passing. CI also runs them with the network off. |
| Results | None yet: not measured in the repo |
| Main blocker | NGX data: the official site blocks automated access |

## Files

| File | What's in it |
|---|---|
| [00-start-here.md](00-start-here.md) | **Start here.** The whole project in simple English (good for NotebookLM) |
| [01-system-design.md](01-system-design.md) | Built vs planned parts, diagram, stack, data flow, trade-offs, 10x |
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
