# Fraud-Detection-With-Sequence-Models

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

## In short

An honest comparison of sequence models (GRU, TCN, Transformer) against a strong LightGBM baseline for real-time card fraud, on a synthetic world with three labelled fraud types and adversarial drift. It includes leakage-tested features shared between training and serving, cost-based decisions, ONNX serving and drift monitoring. The finding: sequence models rank better, and only after the attackers adapt. The cost saving isn't significant.

## Key facts

| | |
|---|---|
| Language | Python, PyTorch, LightGBM |
| Serving | ONNX Runtime + FastAPI, p99 7.2 ms reported |
| Ranking | GRU PR-AUC 0.870 ± 0.014 vs LightGBM 0.826 ± 0.001 |
| Post-drift | 0.765 vs 0.615 |
| Cost | 0.0799 ± 0.0122 vs 0.0860 ± 0.0035 (not significant) |
| Tests (my run) | 144 passing |
| Note | One commit. Results files not committed. Numbers are from the README. |

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
