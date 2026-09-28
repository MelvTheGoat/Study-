# Fantasy-Premier-League (FPL AI Manager)

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

## In short

Two bots play the 2026/27 Fantasy Premier League season from the same predictions:

- **The Manager** keeps one squad all season under the real rules (1 free transfer, −4 hits, £100m, two chip sets).
- **Best XI** rebuilds the best legal squad from scratch every gameweek.

The score gap between them measures what being stuck with past decisions costs. Both are scored with real FPL points against the official gameweek average.

## Key facts

| | |
|---|---|
| Language | Python 3.11 backend, React 18 + Vite frontend |
| Core tech | FastAPI, SQLite, PuLP/CBC (integer linear programming), httpx |
| Runs on | GitHub Actions (scheduler) + GitHub Pages (static site). Docker and Render also supported. |
| Tests | 610 passing, no network needed |
| Results after GW5 | Best XI 322 pts, Manager 268 pts. Beat the average: 2/5 vs 1/5. |
| Not measured yet | Prediction accuracy, and any backtest on a past season |
| Commits | 48, from 11 to 23 Sep 2026 |

## Files

| File | What's in it |
|---|---|
| [01-system-design.md](01-system-design.md) | Problem, diagram, each part, tech stack, data flow, trade-offs, 10x scale |
| [02-how-to-write-the-system-design.md](02-how-to-write-the-system-design.md) | Step-by-step whiteboard guide for this system |
| [03-linkedin-post.md](03-linkedin-post.md) | Short LinkedIn post |
| [04-blog-post.md](04-blog-post.md) | Full blog post with real code snippets |
| [05-explain-to-non-technical.md](05-explain-to-non-technical.md) | Plain-language explanation with an everyday comparison |
| [06-explain-to-technical.md](06-explain-to-technical.md) | Architecture, model, optimiser, tests, weaknesses |
| [07-defend-in-interview.md](07-defend-in-interview.md) | 60-second pitch, 15 questions and answers, weak spots |
| [08-ten-points-to-know.md](08-ten-points-to-know.md) | 10 facts to know by heart |
| [09-what-this-proves-i-know.md](09-what-this-proves-i-know.md) | Skills shown, plus related topics to prepare |

## Honest notes

- The project's own docs are behind the code. The README says ~490 tests and `how-it-works.md` says 574, but 610 pass. The chip thresholds and max transfers in the docs are older values. These reports follow the code.
- The results table in `how-it-works.md` (171 vs 196 after GW4) is out of date. The GW1–4 picks in the live database were re-made on 17 Sep 2026, minutes after the minutes-model fix, so the numbers here come from the `season-data` database on 28 Sep 2026.
- That also means **GW1–4 were picked by a model that was changed after those weeks had been played.** The data used was still pre-deadline, but the model design wasn't blind to those weeks. Only GW5 onward is a truly live test.
