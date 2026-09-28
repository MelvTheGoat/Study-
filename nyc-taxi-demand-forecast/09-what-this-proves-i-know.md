# nyc-taxi-demand-forecast: What This Proves I Know

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

---

## 1. Time-series forecasting

**Simple explanation:** predict future values from past values plus known patterns (time of day, day of week, holidays).

**In this project:** 24h-ahead hourly forecasts per zone, seasonal baselines, a weekly profile model, a global LightGBM.

**Also be ready to explain:** seasonality, trend, level shifts, direct vs recursive multi-step forecasting, global vs local models, and ETS/SARIMA vs ML.

---

## 2. Quantile regression and the newsvendor problem

**Simple explanation:** when over- and under-prediction cost different amounts, predict the quantile that balances them.

**In this project:** pinball loss, τ* = c_u/(c_u+c_o) = 0.75, and empirical operating-quantile selection.

**Also be ready to explain:** the pinball loss formula, prediction intervals, quantile crossing, conformal prediction, and CRPS.

---

## 3. Leakage-safe feature engineering and backtesting

**Simple explanation:** make sure features only use information available at forecast time, and test the way you'll deploy.

**In this project:** origin-relative features with declared source hours, rolling-origin backtesting with an expanding window, and a target-based barrier.

**Also be ready to explain:** rolling vs expanding windows, walk-forward validation, horizon-aware lags, and gap/embargo periods.

---

## 4. Forecast error metrics

**Simple explanation:** measure how wrong forecasts are, in ways that compare across series.

**In this project:** MAE, RMSE, MASE, pinball loss, total business cost, under-forecast share.

**Also be ready to explain:** MASE scaling (why the denominator matters), why MAPE fails on small counts, and scale-free vs scale-dependent metrics.

---

## 5. Data engineering with dbt and DuckDB

**Simple explanation:** transform raw data into clean, tested tables using SQL.

**In this project:** staging/intermediate/marts layers, named cleaning rules, a data-quality mart, generic and singular tests, a holiday seed.

**Also be ready to explain:** dbt models, refs, sources and tests, incremental models, idempotent ingestion, and checksum manifests.

---

## 6. MLOps: tracking, registry, promotion

**Simple explanation:** record every run, and only replace the live model when a new one is clearly better.

**In this project:** MLflow tracking, registry aliases (`@staging`, `@production`), a 2% margin, a CI regression gate.

**Also be ready to explain:** model registries, champion/challenger vs A/B vs shadow, and retraining triggers (schedule vs drift vs performance).

---

## 7. Drift monitoring

**Simple explanation:** notice when the data or the model's accuracy changes.

**In this project:** PSI, the KS test, pooled vs per-zone, rolling MASE, excluding calendar features, Evidently reports.

**Also be ready to explain:** the PSI formula and thresholds, KS statistics, covariate vs concept drift, and Simpson's-paradox-style masking in aggregates.

---

## 8. Orchestration

**Simple explanation:** schedule and chain pipeline steps.

**In this project:** Prefect flows (hourly monitor, daily pipeline).

**Also be ready to explain:** DAGs, retries, idempotency, Airflow vs Prefect vs Dagster, and backfills.

---

## 9. Serving forecasts

**In this project:** FastAPI `/forecast` returning the operating value and quantiles, plus `/health`, `/metrics`, `/model-info`.

**Also be ready to explain:** precomputing vs on-demand forecasts, caching, and exposing uncertainty to downstream decisions.

---

## 10. Synthetic data for testing systems

**Simple explanation:** generate data with known properties (like injected drift) so you can check that the system reacts correctly.

**In this project:** a TLC-schema generator with seasonality, holidays, dirty rows and a zone redistribution drift.

**Also be ready to explain:** the limits of synthetic data (it can't reveal unknown real-world effects), and seeding for reproducibility.
