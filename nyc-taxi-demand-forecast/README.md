# nyc-taxi-demand-forecast

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

## In short

A self-operating system that forecasts hourly NYC taxi pickups per zone, 24 hours ahead, for dispatch. Because under-supply is priced at 3× over-supply, it forecasts quantiles and ships the cost-optimal one (theory says q0.75, and the data chose q0.80), not the mean. It includes leakage-proof origin-relative features, a rolling-origin backtest, dbt + DuckDB, pooled and per-zone drift monitoring, champion/challenger retraining with MLflow, Prefect scheduling and a FastAPI service.

## Key facts

| | |
|---|---|
| Language | Python + SQL (dbt) |
| Core tech | DuckDB, dbt, LightGBM (quantiles), MLflow, Prefect, FastAPI |
| Headline (synthetic) | LightGBM cost 104,568 (−32% vs seasonal naive). The per-zone profile model ties it. |
| Key insight | Shipping the median instead of q0.80 costs +25% |
| Tests (my run) | 167 passing |
| Data | Synthetic only. The real TLC download was blocked. |

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
