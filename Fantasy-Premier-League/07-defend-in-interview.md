# Fantasy-Premier-League: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Fantasy-Premier-League

---

## 60-second pitch (say it out loud)

> "Every Fantasy Premier League manager is stuck with last week's decisions, but nobody can see what that costs, because you only play one team. So I built two bots that play the 2026/27 season from the same predictions. The Manager plays by the real rules: one free transfer, −4 hits, £100m, chips that expire. Best XI rebuilds the perfect squad every week. The gap between them measures the cost of continuity.
>
> Under the hood there's a pure rules engine, an expected-points model built from parts with shrinkage toward price-based priors, and an integer linear program that picks the best legal squad. The database refuses to rewrite a squad after its deadline, so there's no hindsight. It runs for free on GitHub Actions with a stateless scheduler and publishes a static React site to GitHub Pages. There are 610 tests.
>
> After five gameweeks Best XI has 322 points and the Manager 268. That's too early to judge, and the next step is a proper backtest, because the model's accuracy isn't measured yet."

---

## Likely questions and strong, honest answers

### 1. "Why not train a machine learning model to predict points?"
I chose a model built from parts on purpose. Three gameweeks into a season there's very little data, and a trained model would mostly learn noise. Parts also let me explain every pick in football terms, like "home to Coventry, on penalties". The honest downside is that my weights are reasoned, not fitted, and I haven't backtested them yet. A trained model, maybe gradient boosting on past seasons, is a fair next step. I'd only switch if it beat the current model on a held-out season.

### 2. "Why an ILP instead of a greedy pick?"
The constraints interact. Taking the best forward might leave too little for a fifth defender, and the three-per-club limit bites exactly where the best players cluster. Greedy gives something that looks reasonable. The ILP gives the true best answer for the objective. The problem is small, about 667 players with a few dozen constraints, so CBC solves it well under the 30-second limit.

### 3. "How do you know it works?"
Three levels. First, **correctness**: 610 tests, with the densest coverage on the rules and the model, and ingest tests on real recorded API data. Second, **fair scoring**: both teams are scored with real FPL points and compared with the official gameweek average, never with their own predictions. Third, the part I haven't done: **I haven't measured projection accuracy yet.** There's no backtest in the repo. So I can say the system plays legal, rule-correct squads and scores them honestly. I can't yet say the predictions are good.

### 4. "How did you stop data leakage in the backfill?"
The database enforces it rather than relying on me being careful. Prices are stored per gameweek. Projections are keyed by the deadline they were made before. `save_locked_picks` raises `LockedPicksExist` once a deadline has passed, or if there's no deadline recorded. There's one leak I know about: the API doesn't keep history for injury flags, so backfilled weeks see today's team news. I state that in the docs.

### 5. "What would break first?"
Timing. GitHub's cron is best-effort, and I measured gaps of 2 to 7 hours against a 30-minute schedule. A two-hour pick window was missed about two in five times. I widened it to 8 hours and made a run inside the window stay alive until the deadline. If every run is still missed, the backfill rebuilds that gameweek from pre-deadline data. After timing, the next risk is the FPL API changing a field. It has already changed several this season.

### 6. "How would you scale it?"
Visitors scale for free, because it's a static site. If "scale" means picking for many real user teams: compute the shared projection once per deadline, move from SQLite to Postgres for concurrent writes, move jobs from GitHub Actions to a real queue with workers, and run optimiser solves in parallel. I'd also add model versioning with a backtest gate before any new model goes live.

### 7. "Why SQLite?"
One writer, a database measured in megabytes, and one file I can gzip onto a git branch and restore on the next runner. Postgres would add a server I don't need. The trade-off is that only one process can write, and the workflow's `concurrency` setting enforces that.

### 8. "How does the Manager decide on a hit?"
It asks the optimiser for the best squad reachable with 0 to 5 transfers, priced at real selling prices. A hit only counts if the gain beats 4 points plus a 2-point margin, because the predictions are often wrong. Honestly, the comparison uses the optimiser's objective, which includes a small bench-value term, so a hit can be partly justified by a better bench. That's something I'd tighten.

### 9. "How do you choose when to play chips?"
Each chip has its own threshold: Bench Boost 22, Triple Captain 12, Free Hit 18, Wildcard 30. My first version used one shared threshold of 6, and it fired Triple Captain in GW3 and a Wildcard in GW2. A premium captain scores 6–8 in a normal week, so each chip needs a bar where *its* good week starts. There's also no voluntary Wildcard before GW5, and a chip about to expire gets played anyway. These values are corrections, not validated over a season.

### 10. "Why not X instead: a paid server, or a cron on a VPS?"
The site is read-only. Nobody types into it. It needs something that thinks a few times a day and leaves files behind, and GitHub Actions does that for free on a public repo. I do also ship a Dockerfile and a Render blueprint, so moving to an always-on host is one command if the timing risk ever matters more than cost.

### 11. "Tell me about a bug."
The site once published GW1 to GW3 as final at zero points. Nothing in the setup path fetched real results, and scoring a squad against no results doesn't fail. It returns zero. A season of zeros looks like a bad season, not a broken one. I fixed it, and then the fix itself skipped scoring because of a guard that the line above it had changed. The lesson I took: silent wrong output is worse than a crash. I added stale-score detection, setup stages that refuse to publish squads without results, and an end-to-end seed test.

### 12. "Why is the bench weighted at 0.12?"
At zero, the optimiser buys four £4.0m players who never play. That's correct for one week and a disaster when someone's injured. A small weight makes it buy a bench that can cover. There's also a small value-per-million term so it prefers cheap players who are already performing.

### 13. "What's the scheduler's design?"
It's stateless. `due_work(db, now)` is a pure function that reads the database against the clock and returns the jobs owed. A restart, a redeploy or a week offline all resolve the same way. It re-checks up to three times per tick, because one job can create another (a refresh can reveal a finished gameweek to finalise). The clock and sleep are passed in, so time rules are tested without waiting.

### 14. "What are the results?"
From the season database after GW5: Best XI 322, Manager 268, a 54-point gap. The Manager beat the average once in five weeks and Best XI twice. It's too early to conclude anything. I'd rather say neither beats the average reliably yet than dress it up.

### 15. "What would you do next?"
A backtest on the 2025/26 season to measure projection error and tune weights. Recording team news every gameweek to close the leak. Checking blank and double gameweeks against real fixtures. And effective ownership, so the Manager can play against the field.

---

## Weak spots an interviewer might poke at, and how to answer without bluffing

| Weak spot | What they might say | Honest answer |
|---|---|---|
| No backtest | "So you don't know if the predictions are any good?" | "Correct. I know the squads are legal and scored honestly. Prediction accuracy isn't measured yet. A backtest on last season is the first thing I'd add." |
| Hand-set constants | "Where do 0.82 decay and 450 minutes come from?" | "They're reasoned, not fitted. The code comments say so. They're named constants, so a backtest can tune them." |
| Results below average | "Your bots don't beat the average." | "Not reliably yet, after five weeks. The project measures the gap between the two models, and beating the field is a separate goal. Effective ownership is the missing piece for that." |
| Team-news leak | "Isn't the backfill cheating?" | "Partly, on injury flags only. Prices, results and projections are time-boxed. I documented the leak rather than hiding it." |
| Docs out of date | "Your README says 490 tests and different chip thresholds." | "Fair. The code moved on after the docs. 610 tests pass today and the thresholds are 22/12/18/30. I should update the docs." |
| Hit test uses objective | "Your hit rule includes bench value." | "Yes. I'd change it to compare pure projected points over the horizon." |
| Single writer | "What if two runs overlap?" | "The workflow's concurrency group stops that. Beyond one writer I'd move to Postgres." |
| Chip thresholds | "Aren't those just guesses?" | "They're corrections to a clear failure, and they moved when the minutes model changed. Validating them needs a full season or a backtest." |
| Unused constant | "What's `BENCH_SLOT_WEIGHTS` for?" | "It's left over. It's defined but not used. I'd either use it for per-slot bench weights or delete it." |

**Golden rule:** if you don't know a number, say "that isn't measured in the repo yet" and say how you'd measure it.
