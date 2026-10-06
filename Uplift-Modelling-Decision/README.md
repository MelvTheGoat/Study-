# Uplift-Modelling-Decision

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

## In short

A budget-constrained uplift (causal) study on the Hillstrom randomised e-mail trial: who converts *because* of the campaign. Estimators (S/T/X-learners, causal forest) are validated on synthetic data with a known effect, then evaluated on real data with Qini curves (no AUC), priced as a budget frontier, stress-tested (placebo, balance, costs, seeds), and written up as a two-page decision memo. The key honest finding: the men's campaign targeting gain failed the placebo test. Raising the budget is worth more than better targeting.

## Key facts

| | |
|---|---|
| Language | Python (numpy, pandas, scikit-learn, LightGBM, EconML) |
| Data | Hillstrom 2008 (64k customers, 42,613 analysed) + synthetic |
| 30% budget | $3,066 profit (CI $782–$5,535) vs $1,674 random |
| Levers | Budget ≈ $3,170 vs targeting ≈ $1,390 |
| Stress tests | Men's placebo failed. Women's passed. |
| Tests (my run) | 136 passing |
| Note | One commit. Results committed in `results/`. |

## Files

| File | What's in it |
|---|---|
| [00-start-here.md](00-start-here.md) | **Start here.** The whole project in simple English (good for NotebookLM) |
| [01-system-design.md](01-system-design.md) | Pipeline, diagram, stack, data flow, trade-offs, 10x |
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
