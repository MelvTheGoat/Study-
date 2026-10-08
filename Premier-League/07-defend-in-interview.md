# Premier-League: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Premier-League

---

## 60-second pitch

> "I built a Premier League predictor that treats context as data, not rules. Instead of 'new manager, add 5%', manager tenure, congestion, table pressure, squad ratings and so on become 200+ features, built in one strictly chronological pass so a feature can never see the future. Outcomes come from a blend of LightGBM and a multinomial logistic regression, which keeps probabilities calibrated early in the season. Scorelines come from a Dixon-Coles Poisson model.
>
> Promoted clubs are the classic cold-start problem, so Elo is rated across the Premier League, Championship, League One and the cups on one scale. It retrains after every gameweek, never overwrites a forecast, and a daily GitHub Action publishes a Flask site on Vercel and checks the live page. In a walk-forward backtest over 1,050 matches it gets 52.1% accuracy vs 43.2% for always-home, with log loss below an Elo baseline and good calibration."

---

## Questions and honest answers

### 1. "Why not hand-tune adjustments? Pundits know things."
Because hand rules can't be tested and don't adapt. Turning each idea into a measurable feature lets the model decide how much it matters, including "not at all". Where a signal is hard to measure, like a new-manager bounce, I use proxies: days in charge, matches in charge, and the change in points per game.

### 2. "How do you prevent leakage?"
Structurally. Club state only moves forward in one chronological pass. Every match in gameweek N uses the state from before N's first kick-off, so a Monday game doesn't see Saturday's results. Training uses the same rule, so train and serve match.

### 3. "Why blend LightGBM with logistic regression?"
They fail differently. The booster finds interactions but is over-confident with thin early-season data. The linear model can't invent interactions, so it's steadier and better calibrated. The site shows probabilities, so calibration matters. The blend is 0.6/0.4.

### 4. "Why optimise log loss and not accuracy?"
Because the product is probabilities. Accuracy-tuned models stop predicting draws altogether and are over-confident. Log loss rewards honest probabilities.

### 5. "Your model never predicts a draw?"
Correct, as the argmax. A draw is rarely the single most likely outcome. The model puts about 25–30% on draws and the site shows the full three-way split. Forcing draw picks would help draws and hurt everything else.

### 6. "How do you handle promoted clubs?"
Cross-division Elo on one scale, carried between seasons with 25% regression to the mean and division offsets. For scorelines, the Dixon-Coles model is also fitted on Championship results with a level offset, so a promoted club isn't judged on three Premier League games.

### 7. "How do you know it works?"
A walk-forward backtest that replays the exact live procedure: 1,050 matches, 52.1% vs 43.2% always-home, log loss 0.9948 vs 1.0061 Elo-only, and a reliability table where 65% predictions happen about 65% of the time. Live this season, it's 21/50 after five gameweeks, but only one of those gameweeks was published before kick-off, so I don't lean on it yet.

### 8. "What's the weakest part?"
Injuries. The model still can't see team news. FPL availability is now recorded daily, but a few weeks of history isn't enough to learn from. Manager history now comes from Wikidata, but it's thinner before 2016. And there's no xG: I tested it, and it wasn't measurably better.

### 9. "How do you handle data that arrives late?"
Any feature populated for fewer than half the fixtures being predicted is dropped from that run's model and logged. It comes back when the source catches up. Squad ratings are carried forward, with their age as a feature.

### 10. "Why is the most likely score sometimes different from the predicted winner?"
A home win's probability is spread across many scores, so 1-1 can be the single likeliest score while a home win is the likeliest result. The site shows the most likely score consistent with the predicted outcome, with expected goals alongside.

### 11. "Tell me about a production bug."
After the default branch was renamed, the daily job kept pushing and passing while the host deployed the old branch. Eleven green runs, and the site was two gameweeks stale. Now the job fast-forwards other deploy branches (never force) and ends by fetching the live site and failing if it's not showing the right gameweek.

### 12. "How would you scale to more leagues?"
Parameterise the league in config and sources, build features incrementally rather than fully each run, move features to a warehouse and artefacts to object storage, and orchestrate per-league jobs.

### 13. "Why SQLite committed to the repo?"
The site is read-only and needs 1.4 MB of data. Shipping it with the function means no database server and free hosting, and each commit is a versioned snapshot of the public record.

### 14. "What's Dixon-Coles?"
A Poisson goals model with attack and defence ratings per team and a home advantage, plus a correction term (rho) for low scores, because independent Poisson gets 0-0, 1-0, 0-1 and 1-1 wrong. I add exponential time decay and ridge regularisation.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Backtest numbers from README | "Did you verify?" | "They come from the repo's walk-forward script. The live record is separate and small." |
| Live 42% | "That's below the backtest." | "50 matches, mostly backfilled. The noise is huge at that size. I'd judge after a full season." |
| Backtest starts at GW4 | "You skipped the hard weeks." | "Yes. The first weeks have little in-season data. I'd report them separately." |
| No injuries | "Team news is everything." | "Agreed. FPL keeps no history, so I now record availability every day. Once there's enough of it, it can be tested as a feature." |
| Postponed matches | "What about rearranged fixtures?" | "Matches now go in the gameweek they're played in. Before that fix, 258 results leaked into earlier features. Now it's 6, none more than four days." |
| Doc drift | "216 or 213 features?" | "The docs differ. The feature set changes as sources come in. The run record lists the exact columns used." |
