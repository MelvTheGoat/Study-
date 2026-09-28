# nyc-taxi-demand-forecast: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

1. **Forecasts hourly taxi pickups per NYC zone, 24h ahead, for dispatch, and runs itself end to end from one command.**
   *Why it matters:* a system, not a notebook.

2. **Cost = 3·under + 1·over → pinball loss → optimal quantile τ* = 0.75.**
   *Why it matters:* the core insight: forecast a quantile, not the mean.

3. **The operating quantile is chosen on backtest cost: q80 won. Shipping q50 costs +25%.**
   *Why it matters:* the decision rule is worth as much as the model.

4. **Features are relative to the forecast origin, each declares its newest source hour, and `TargetLag(k)` is only legal if k ≥ horizon.**
   *Why it matters:* prevents the classic `lag_1h` leak.

5. **Four leakage tests (contract, future perturbation, future-shock correlation, no-random-split scan) plus a target-based fold barrier.**
   *Why it matters:* leakage fails silently.

6. **Results (synthetic): LightGBM cost 104,568 (−32% vs seasonal naive). The per-zone profile model ties it (104,782).**
   *Why it matters:* simple baselines deserve a fair fight.

7. **dbt + DuckDB: staging → 9 named cleaning rules → hourly marts, with custom tests (no grid gaps, lag alignment).**
   *Why it matters:* tested data engineering.

8. **Drift: PSI + KS on demand features, pooled AND per zone. Pooled flagged 2/16, per zone 8/8.**
   *Why it matters:* aggregates hide redistribution.

9. **Champion/challenger with a 2% margin via MLflow aliases. Prefect schedules. FastAPI serves operating + q10–q90.**
   *Why it matters:* safe automation.

10. **Limits: synthetic data (TLC blocked), no weather/events, an assumed 3:1 ratio, independent zones, single node.**
    *Why it matters:* say these before you're asked.
