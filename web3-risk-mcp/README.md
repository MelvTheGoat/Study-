# web3-risk-mcp

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

## In short

A read-only MCP server that lets any AI assistant check a crypto wallet, token or smart contract for scam risk across 5 EVM chains, and returns a 0–100 score with a reason for every point. Analysis produces findings. A separate, public rule table turns them into a score.

## Key facts

| | |
|---|---|
| Language | Python 3.11+ |
| Core tech | MCP SDK, httpx (async), pydantic, Etherscan V2, GoPlus, DexScreener, RPC |
| Scoring | 85 rules, decisive floor 75, confidence from source success |
| Tests | 102 passing |
| Evaluation | 34 labelled addresses. Live results not measured yet. |

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
