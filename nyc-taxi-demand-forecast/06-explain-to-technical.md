# nyc-taxi-demand-forecast: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

---

## Summary

A self-operating hourly demand forecasting system for NYC taxi zones, 24 hours ahead: idempotent TLC ingestion with a synthetic fallback, dbt + DuckDB transforms with 9 named cleaning rules and custom tests, origin-relative features with a declared leakage contract, a rolling-origin backtest, a LightGBM quantile family chosen by a 3:1 asymmetric cost, MLflow tracking with alias-based promotion, pooled and per-zone drift monitoring, Prefect orchestration, and a FastAPI service. ~7,100 lines of Python. The main build was 3–6 Aug 2026, after an early June scaffold. **Results are synthetic.**

## Architecture

```
src/synthetic.py    TLC-schema generator: seasonality, holidays, drift (zones +38% / −26%), dirty rows
src/ingest.py       downloads with tenacity retries, checksum manifest, null profiling, fallback
dbt_project/        staging -> intermediate (9 rules) -> marts (hour x zone grid, lags, rollings) + DQ mart
src/transform.py    dbt runner + warehouse reads (DuckDB)
src/features.py     OriginLag / OriginRolling / TargetLag / CalendarFeature + newest-hour contract
src/models.py       seasonal naive, hour-of-week mean, SeasonalProfileZone, LightGBM quantiles, cost-weighted LGBM
src/cost.py         asymmetric cost, optimal_quantile (newsvendor), select_operating_quantile, pinball
src/backtest.py     rolling origin, expanding, 6 folds, target-before-test-origin barrier
src/tracking.py     MLflow tracking + registry aliases
src/monitoring.py   PSI + KS (pooled + per zone) on demand features, rolling MASE, Evidently report, decision
src/retrain.py      champion vs challenger, 2% margin
src/regression_gate.py + baselines/  CI metric gate (10% tolerance)
src/flows.py        Prefect: hourly monitor, daily pipeline
src/serve.py        FastAPI: /forecast /health /metrics /model-info
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| Quantile loss from business cost | 3:1 → τ* = 0.75 | Pinball loss = asymmetric linear cost |
| Pick the operating quantile on data | q80 won (theory q75) | Exposes quantile miscalibration as a shifted operating point |
| Origin-relative features | Each declares its newest source hour | `lag_1h` leaks at a 24h horizon |
| `TargetLag(k)` only if k ≥ horizon | — | Seasonal lags that stay legal |
| Target-based fold barrier | Train rows' targets < first test origin | Filtering on origin alone leaks |
| No random splits, enforced by a code scan | Tokenised scan for `train_test_split`, `KFold`, `shuffle=True` | Structural guard |
| Monitor demand features only | Calendar excluded | Calendar PSI 5–9 on a healthy pipeline |
| Pooled + per-zone drift | Alarm on either | Pooled flagged 2/16, per-zone 8/8 |
| 2% promotion margin | MLflow aliases | Don't promote noise |
| Different MASE denominators | Backtest: per-fold train. Monitor: fixed reference. | Each is right for its purpose. Not comparable to each other (documented). |
| dbt + DuckDB | Local SQL with tests | Free, reproducible |

## Models

- **Seasonal naive:** y[t − 168h].
- **Hour-of-week mean.**
- **`SeasonalProfileZone`:** `profile[zone, how(target)] × (mean last 168h / mean profile)`.
- **LightGBM global quantile family:** one model per τ in {0.1 … 0.9}, `zone_id` as a categorical.
- **Cost-weighted LightGBM:** custom asymmetric objective with the correct argument order and an `init_score` at the unconditional τ* quantile. Correlates 0.99 with q75, but 3% worse on cost.

## Evaluation

**Headline (README, synthetic: 6 folds × 7 days, 24h, 8 zones, ~32k zone-hours):**

| Model | MAE | RMSE | MASE | Op. q | Cost |
|---|---|---|---|---|---|
| LightGBM global | 1.861 | 2.600 | 0.825 | 0.80 | 104,568 |
| Seasonal profile zone | 1.846 | 2.579 | 0.818 | 0.75 | 104,782 |
| Cost-weighted LGBM | 2.117 | 2.762 | 0.938 | 0.75 | 108,648 |
| Hour-of-week mean | 1.948 | 2.780 | 0.863 | 0.80 | 113,594 |
| Seasonal naive | 2.352 | 3.259 | 1.042 | 0.60 | 154,010 |

- Operating-point sweep (LightGBM): q50 130,341 (+25%), q75 105,557 (+1.2%), q80 104,265, q90 110,060.
- The committed CI baseline (`baselines/backtest_baseline.json`, a smaller CI config: 3-day folds, stride 12h) has LightGBM MASE 0.863 and cost 9,634 vs seasonal naive 14,512. It's a different config, so it's not comparable to the headline table.
- **Tests:** 167 pass (I ran them, ~2 min 49 s), matching the README.

## Known weaknesses

- **Synthetic only.** The TLC endpoint was blocked. Real data has weather, events and fatter tails.
- **Small counts per zone-hour** (a Poisson noise ceiling).
- **No exogenous regressors** (weather, events).
- **The 3:1 ratio is assumed.**
- **Marginal quantiles** (not joint across hours).
- **No shadow deployment.** Promotion is one-shot.
- **Independent zones.** No cross-zone structure.
- **No data versioning.**
- **Single node** (DuckDB, one API replica, in-process metrics).
- **Retrain pacing:** a duel with no new data is meaningless (17,064 vs 17,064).
- **Repo hygiene:** a stray `Startup_Analysis.ipynb` from another project sits in the root.
