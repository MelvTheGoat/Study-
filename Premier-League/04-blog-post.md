# Predicting the Premier League without a single hand-made rule

Repo: https://github.com/MelvTheGoat/Premier-League

## Why I built it

A scoreline and a league table describe what happened. They don't describe the situation a match was played in, and the situation often decides it.

A club four days after a European away tie fields a different side. A club that changed manager last month is a different team from the one that finished last season. A club with nothing left to play for in May isn't the club that was chasing a European place in February.

The easy way to handle this is a checklist: "if new manager, add 5%". I deliberately didn't do that. **Every contextual signal becomes a number in the feature table, and the model decides what it's worth.** Nothing anywhere applies a hand-tuned adjustment to a prediction.

The result is a system that predicts every Premier League match each gameweek, retrains after every gameweek, and keeps a public record of every forecast it made.

## The problem

Four things had to be true:

1. **No leakage.** The model predicting gameweek N must learn only from matches before gameweek N.
2. **Context as features, not rules.** Where a signal resists measurement, find a better proxy.
3. **Honest probabilities.** The site publishes probabilities, so they have to be well calibrated.
4. **A public record that can't be rewritten.** A forecast made before kick-off must stay exactly as it was made.

## How it works

### Data

Results and fixtures come from `openfootball/england`, a public git repo covering the Premier League, Championship, League One and both domestic cups. It's free and current. Match statistics (shots, cards, corners, referee) come from another public dataset, but it lags the live season. Squad quality comes from FIFA/EA FC ratings, aggregated offline into one row per club per season. A few small hand-maintained files cover manager spells, European participation, club-name aliases and injuries (that last one ships empty).

### Features: context turned into numbers

Every match gets 200+ columns, each computed for the home side, the away side and the difference:

| Family | Examples |
|---|---|
| Strength | Cross-division Elo, Elo win expectation |
| Form | Points, goals, shots, win and clean-sheet rate, and the opposition's average Elo, over the last 4, 6 and 10 matches across all competitions |
| Table context | Position, points per game, and gaps to first, top four, top six and the relegation zone |
| Congestion | Days of rest, matches in the last 14/21 days, days to the next game, midweek flag, European tier |
| Manager | Days and matches under the current manager, and points-per-game change vs the previous one |
| Squad quality | Rating standardised within season, attack/midfield/defence units, rating age |
| Head-to-head, venue, season shape | Last six meetings, home/away records, gameweek, fraction of season left |

"New manager bounce" isn't a lookup. It becomes three measurable things: days since appointment, matches under the new manager, and points per game before vs after. If they matter, the model finds them. If not, it ignores them.

### No leakage by construction

The feature table is built in **one chronological pass**. Club state only moves forward, so a feature *can't* see a result that hasn't been fed in yet. The guarantee is structural, not a filter someone has to remember.

One subtle detail: every match in gameweek N uses the state from **before that gameweek's first kick-off**, so a Monday game doesn't get Saturday's results. That's exactly what's known when the forecast is published, so training and serving see the same information.

### Two models

**Outcome (home/draw/away):** a blend of a LightGBM gradient-boosted classifier over all features and a multinomial logistic regression over a compact core of strength and form features.

```python
weight = self.config.blend_weight
blended = weight * boosted + (1 - weight) * linear
return blended / blended.sum(axis=1, keepdims=True)
```

The blend weight is 0.6. The booster finds interactions, like congestion hurting more when key players are missing, but it's over-confident when a season is young. The linear model can't invent interactions, which makes it better calibrated early. Training optimises log loss, and the number of boosting rounds is chosen by early stopping on a chronological tail of the training data at every retrain.

**Scoreline:** a Dixon-Coles Poisson goal model. Each club has an attack and a defence strength. Home expected goals = home attack × away defence × home advantage. On top of that there's a correction for low scores (plain Poisson under-predicts 0-0 and 1-1), time decay so older matches count less, and **Championship results fitted too**, with a level offset.

## The hard parts

### Promoted clubs

Three clubs a season arrive with no Premier League form, no table position and no useful head-to-head. Left alone, they're blanks for two months.

The fix is that **Elo is rated across divisions on one scale**: Premier League, Championship, League One and the cups. Ratings carry between seasons, regressed 25% toward the mean because squads turn over, with a starting offset by division. A promoted club brings a full season of results against opposition whose strength is known.

The scoreline model had the same problem separately. A promoted club with one goal in three games looked incapable of scoring. Fitting Championship results too, with clubs moving up and down tying the scales together, fixed it.

### Sources that lag

Match statistics arrive weeks late. A feature that exists for every training row but is missing for the gameweek being predicted is worse than no feature. So any column populated for fewer than half the fixtures being predicted is **dropped from that run's model**, and the run records which columns it dropped. The column comes back on its own when the source catches up.

Squad ratings are published once a season, well into it. So the latest rating is carried forward and its **age in seasons** is its own feature. The model can discount a stale rating instead of being handed a silent lie.

### Scorelines that look broken

The most likely *score* and the most likely *outcome* often disagree. A home win's probability is spread across 1-0, 2-0, 2-1 and more, so 1-1 can be the single likeliest score even when a home win is the likeliest result. Showing "home win, 1-1" would look broken. So the site shows the most likely score *consistent with the predicted outcome*, with the raw expected goals alongside. Exact scores are never marked right or wrong. Only outcomes get a ✓ or ✗.

### Eleven green runs on a stale site

The daily GitHub Actions job pushed to the default branch. Then the default branch was renamed, and the host kept deploying the old one. The job went on reporting success. **Eleven consecutive green runs, and a site stuck two gameweeks back.**

Nothing inside the repo could catch that, because from inside everything worked. So now:
- after pushing, the job **fast-forwards** the same commit onto any other branch that's strictly behind (no `--force` anywhere), and
- every run ends by **asking the live site**: `check_publication.py` confirms every played gameweek has a forecast of record, and `check_live_site.py` fetches the deployed page and fails unless it shows the right gameweek.

A failing scheduled workflow emails the owner. Silence now means healthy.

## What it's worth

From the repo's walk-forward backtest (predicting each gameweek knowing only what came before), over 2023-24 to 2025-26 from gameweek 4, 1,050 matches:

| Metric | Model | Baseline |
|---|---|---|
| Outcome accuracy | 52.1% | 43.2% (always home win) |
| Log loss | 0.9948 | 1.0061 (Elo only) |
| Exact scoreline | 8.2% | — |

Calibration is good: when the model says 65%, it happens about 65% of the time. Draws are almost never the pick. They show up as 25–30% probability instead.

The live 2026-27 record so far, from the committed database: after five gameweeks, 21 of 50 outcomes are right (42%), with a log loss of 1.050. But only gameweek 4 was published fully before kick-off (the first three were backfilled when the site launched), and 50 matches is far too few to say anything.

## What I learned

- **Features, not rules.** Turning intuition into measurable proxies keeps you honest: the model is allowed to disagree with you.
- **Leakage prevention should be structural.** A forward-only state machine beats a filter you have to remember.
- **Calibration matters more than accuracy** when you publish probabilities.
- **Don't trust your own green CI.** Check the thing users see.

## What's next

The repo lists the gains in the order most likely to pay: automated injury and team news, older manager history, an expected-goals feed, and more timely match statistics.
