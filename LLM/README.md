# LLM (gptlab)

Repo: https://github.com/MelvTheGoat/LLM

## In short

A from-scratch PyTorch system that trains small GPT-style models (~1M–100M parameters) on free Kaggle GPUs, with exact stop-and-resume across 11-hour sessions and a job queue for experiments (scaling, ablations, stability, efficiency, evaluation). **All code is built and tested on CPU. No GPU runs yet, so no results.**

## Key facts

| | |
|---|---|
| Language | Python, PyTorch |
| Data | FineWeb-Edu sample-10BT, 16k byte-level BPE, uint16 shards |
| Hardware | Kaggle 2× T4 (fp16) |
| Tests | 170 passing on CPU (~54 s) |
| Results | Not measured yet (no `results` branch) |

## Files

| File | What's in it |
|---|---|
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
