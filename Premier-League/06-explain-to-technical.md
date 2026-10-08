# Premier-League: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Premier-League

---

## Summary

A Premier League match predictor with a strictly chronological feature pipeline (~200+ features: cross-division Elo, rolling form, table gaps, congestion, manager tenure, squad ratings, H2H, venue, season shape), a 0.6/0.4 blend of LightGBM and multinomial logistic regression for outcomes, a Dixon-Coles Poisson model for scorelines, an append-only prediction store, and a Flask site served from a committed read-only SQLite file on Vercel. A daily GitHub Actions job retrains only when something changed and verifies the live site. **67 tests pass (~9 s).** ~5,000 lines. 20 commits, 10–23 Sep 2026.

## Architecture

```
plpredict/
  config.py            paths, seasons, FeatureConfig, OutcomeModelConfig, ScorelineModelConfig
  db.py                SQLite schema
  data/sources/        openfootball (primary), footballdata (stats), fifa_ratings
  data/teams.py        name normalisation
  data/ingest.py       sources -> DB
  features/elo.py      cross-division Elo
  features/state.py    forward-only per-club state
  features/build.py    chronological pass -> versioned feature table (JSON per match)
  models/outcome.py    LightGBM + multinomial LR blend, usable_columns
  models/scoreline.py  Dixon-Coles with decay, ridge, Championship offset
  models/evaluate.py   walk-forward backtest
  pipeline/run.py      rolling retrain loop, model_runs, history restore
  web/                 Flask app, queries, export, templates
scripts/               CLI wrappers, backtest, checks
.github/workflows/update-predictions.yml
```

**Tables:** `matches`, `match_stats`, `managers`, `unavailability`, `squad_ratings`, `european_participation`, `features`, `model_runs`, `predictions`, `current_predictions`.

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| No hand-made adjustments | Every signal is a feature | The model weighs evidence, and can ignore weak signals |
| Gameweek cutoff | All matches in GW N use the state before N's first kick-off | Train/serve parity, no within-gameweek leakage |
| Forward-only state | Single chronological pass | Leakage prevention is structural |
| Home/away/diff for every family | Explicit difference columns | Saves the model learning subtraction |
| Cross-division Elo | K=20, home +60, 25% regression, offsets Champ −180 / L1 −320, initial 1500 | Promoted clubs |
| Outcome blend | `0.6·LGBM + 0.4·LR`, renormalised | Interactions + early-season calibration |
| LGBM settings | lr 0.04, 16 leaves, min_child 60, subsample 0.85, colsample 0.7, λ 5, early stop 60 on a 15% chronological tail, 40–1,200 rounds | Regularised for small, noisy data |
| Objective | Log loss | Probabilities are the product |
| Dixon-Coles | Decay 0.0011/day (half-life ~630 days), ridge 0.35, max 8 goals, ≤1,500 days of history, PL + Championship | Goals are counts. Stable ratings for promoted sides. |
| Coverage gate | Drop columns <50% populated in the target gameweek (and <50 non-null in train) | Lagging sources |
| Rating age feature | Carry the last rating forward, plus age in seasons | Stale data made explicit |
| Append-only runs | Each retrain adds `model_runs` + predictions. `current_predictions` points to the record. | Forecasts can't be rewritten |
| History restore | Rebuilt DB restores published history before predicting | A rebuild can't erase the record |
| Late flag | Predictions stored after kick-off are marked | Honesty about advance vs late |
| Serving DB | ~1.4 MB export, no WAL, read-only | Serverless-friendly |
| Scoreline display | Most likely score consistent with the predicted outcome | Avoids "home win, 1-1" |

## How it's evaluated

**Backtest (numbers reported in the repo's README, from `scripts/backtest.py`; I didn't re-run it):** walk-forward over 2023-24, 2024-25 and 2025-26 from GW4, 1,050 matches.

| Metric | Model | Baseline |
|---|---|---|
| Accuracy | 52.1% | 43.2% always-home |
| Log loss | 0.9948 | 1.0061 Elo-only |
| Brier | 0.5932 | — |
| Exact score | 8.2% | — |
| Goals MAE | 0.93 per side | — |

Per season: 57.1% / 52.0% / 47.1%. The reliability table shows predicted vs observed within about 0.04 across the bins (e.g. 0.65 → 0.65, 0.74 → 0.70).

**Live 2026-27 (computed by me from the committed serving DB):** GW1–5, 50 matches, 21 correct (42.0%), log loss 1.050, 3 exact scores (6%). The model never picked a draw. Only GW4's run was stored before its first kick-off. GW1–3 were backfilled on 10 Sep, and GW5 was stored the morning after its Friday opener.

**Tests:** 98 in total: 97 pass in my run and 1 fails (`test_a_prediction_stored_after_kickoff_is_flagged`, which asserts the upcoming gameweek isn't yet late). Areas: features, models, the openfootball parser, pipeline + web, publication checks, teams, gameweek assignment, Wikidata managers, FPL snapshots.

**Tested and not shipped (Oct 2026, README):** walk-forward over 2019-20 to 2025-26 (2,660 matches), paired bootstrap on log loss. The live version scored 0.9889 log loss and 52.4% accuracy. The gameweek fix, Understat xG, an injury proxy, Wikidata managers and a long-memory xG rating each moved log loss by −0.0013 to +0.0006, and every 95% interval crossed zero.

## Known weaknesses

- **Injuries absent from the model:** the FPL availability log (since 30 Sep 2026, changes only) has too little history to train on.
- **Manager coverage thinner before 2016:** Wikidata gives 84% of club-matches overall, 57–68% for 2010–2015.
- **Gameweek leak mostly closed:** `gameweeks.py` reassigns matches to the gameweek they're played in (190 moved). Future-result leaks fell from 258 to 6, none more than four days.
- **No xG.** Shots on target is the proxy, and stats lag the season, so the live model is thinner than the backtested one.
- **Draw handling:** argmax never picks a draw. That's fine for log loss, but it looks odd to users.
- **Backtest from GW4 onward** excludes the hardest early weeks.
- **Live record is tiny and mostly late.** Nothing to conclude yet.
- **Full rebuild every run**, and PL-specific config.
- **Doc drift:** README 216 features vs build doc 213. Build doc 55 tests vs 98 actual.
