# nyc-taxi-demand-forecast: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

---

## Step 1: Requirements (2 min)

One line:
> "Forecast hourly taxi pickups per zone 24 hours ahead, so dispatch can position drivers at the lowest cost, and keep the system healthy on its own."

**Functional**
1. Ingest trip data on a schedule.
2. Clean it and aggregate to an hour × zone grid.
3. Forecast 24h ahead with quantiles.
4. Pick the forecast to act on, using the cost of under- vs over-supply.
5. Monitor drift and error, and retrain and promote safely.
6. Serve forecasts via an API.

**Non-functional**
- **No leakage** in features or backtests.
- **Idempotent** ingestion.
- **Safe promotion:** a challenger must clearly win.
- **Reproducible:** one command, tracked runs.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Horizon | 24 hours |
| Zones in the headline run | 8 (synthetic) |
| Backtest | 6 folds × 7-day windows, ~32k zone-hours per model |
| Cost ratio | under : over = 3 : 1 → τ* = 0.75 |
| Quantiles | q10 … q90 |
| Promotion margin | 2% |
| Drift monitoring | PSI + KS, pooled + per zone |

**Say:** "The data is small (hours × zones). The hard parts are leakage, choosing the right quantile, and safe automation."

---

## Step 3: High-level boxes (2 min)

```
[TLC parquet / synthetic] -> [Ingest + checksum] -> [dbt: staging -> 9 rules -> hourly marts] (DuckDB)
        -> [Origin-relative features] -> [Rolling-origin backtest] -> [Cost: pick quantile]
        -> [MLflow registry: @production]
[Monitor: PSI/KS pooled + per zone, MASE] -> breach? -> [Challenger duel, 2% margin] -> promote/hold
[FastAPI /forecast] <- @production model + marts        [Prefect schedules everything]
```

---

## Step 4: Deep dive (10 min)

### 4a. The cost framing (start here)
- `cost = 3·max(y−ŷ,0) + 1·max(ŷ−y,0)` is the pinball loss, so the optimum is the **τ = 3/(3+1) = 0.75 quantile**.
- So fit quantiles, not the mean, and pick the operating quantile from backtest cost (q80 won empirically).

### 4b. Leakage-proof features
- Define every feature relative to the **forecast origin** (the last observed hour).
- Each feature declares its newest source hour. `TargetLag(k)` is allowed only if `k ≥ horizon`.
- Tests: contract check, perturb the future and expect no feature to change, future-shock correlation, and a scan for random splits.

### 4c. Backtest
- Rolling origin, expanding window, 6 folds.
- Admit a training row only if its **target** is before the first test origin.

### 4d. Models
- Baselines: seasonal naive, hour-of-week mean.
- `SeasonalProfileZone`: weekly profile × recent-level ratio. Tracks level shifts instantly.
- LightGBM quantile family (one model per quantile).

### 4e. Monitoring and retraining
- PSI + KS on demand features only (calendar features "drift" by design).
- **Per zone as well as pooled:** pooled drift missed a redistribution (2/16 flagged pooled vs 8/8 zones).
- Rolling MASE.
- Challenger must beat the champion by 2% on the same backtest. MLflow aliases.

### 4f. Serving
- `/forecast` returns `operating` (the cost-optimal quantile) plus q10/q50/q75/q90.

---

## Step 5: Bottlenecks (2 min)

1. **Data arrival pace:** a challenger trained on no new data just ties (seen: 17,064 vs 17,064).
2. **Drift masking** in aggregates. Handled with per-zone checks.
3. **Missing exogenous signals** (weather, events).
4. **Single node.** Fine now.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Quantile regression | Mean/MAE | 3:1 asymmetric cost | Several models |
| Empirical quantile pick | Fixed 0.75 | Catches miscalibration (q80 won) | Needs backtests |
| Profile model kept | Drop simple model | Ties LightGBM | Two paths to maintain |
| Per-zone drift | Pooled only | Pooled missed the drift | More alarms |
| 2% promotion margin | Any improvement | Avoid promoting noise | Slower adoption |
| DuckDB + dbt | Cloud warehouse | Local, free | Single node |
