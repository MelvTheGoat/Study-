# Premier-League (Premier League Predictor)

Repo: https://github.com/MelvTheGoat/Premier-League

## In short

Predicts every Premier League match each gameweek (home/draw/away probabilities and a scoreline) from 200+ context features, retrains after every gameweek, keeps an append-only public record, and publishes a Flask site on Vercel via a daily GitHub Action that checks the live page.

## Key facts

| | |
|---|---|
| Language | Python |
| Models | 0.6 LightGBM + 0.4 multinomial logistic (outcome), Dixon-Coles Poisson (scoreline) |
| Key idea | Context as features, not rules. Cross-division Elo for promoted clubs. |
| Backtest (README) | 1,050 matches: 52.1% accuracy vs 43.2% always-home, log loss 0.9948 vs 1.0061 Elo |
| Live 2026-27 | GW1–5: 21/50 correct. Only GW4 was published before kick-off. |
| Tests | 67 passing |

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
| [13-talking-it-through.md](13-talking-it-through.md) | The whole build talked through like a conversation, start to finish |
