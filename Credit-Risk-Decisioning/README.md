# Credit-Risk-Decisioning

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

## In short

A credit decisioning system, not a default-prediction notebook. It produces calibrated probabilities of default, approves or declines at the lowest expected cost (`p* = margin/(margin+LGD) ≈ 0.107`), explains declines with adverse-action reason codes, audits fairness against ground truth, monitors drift, and serves decisions through one engine (FastAPI + batch) with an append-only audit log. It's built and validated on a synthetic loan book with known outcomes for everyone.

## Key facts

| | |
|---|---|
| Language | Python 3.11 |
| Core tech | LightGBM, scikit-learn, statsmodels, SHAP, FastAPI, Streamlit, Docker, AWS ECR |
| Headline (synthetic, out-of-time) | LightGBM net cost −2,506,563. F1 cutoff costs 729,735 more. A 0.5 cutoff loses money. |
| Fairness | AIR 0.755, and 2.0 points of manufactured PD excess from income bias |
| Tests (my run) | 227 pass with statsmodels <0.15. 4 Heckman tests fail on statsmodels 0.15. |
| Real data | UCI path implemented, not run on the real file |

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
