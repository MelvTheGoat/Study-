# nyc-taxi-demand-forecast: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

---

## 60-second pitch

> "I built a self-operating system that forecasts hourly taxi pickups per NYC zone, 24 hours ahead, for dispatch. The key design choice comes from the business: under-supply costs about three times over-supply, which makes the cost a pinball loss, so the optimal forecast is the 75th percentile, not the mean. I fit a LightGBM quantile family and pick the operating quantile from backtest cost. Shipping the median instead would cost 25% more.
>
> Features are defined relative to the forecast origin, with a declared leakage contract and four tests that attack it, and the backtest is rolling-origin with a target-based barrier. Drift is monitored pooled and per zone, because pooled checks missed a redistribution that per-zone checks caught in all 8 zones. A challenger must beat the champion by 2% before MLflow promotes it. The whole thing runs with dbt, DuckDB, Prefect and FastAPI. Caveat: the results are on synthetic data, because the TLC download was blocked where I built it."

---

## Questions and honest answers

### 1. "Why forecast a quantile instead of the mean?"
Because the costs are asymmetric. With under at 3 and over at 1, the expected cost is minimised at the τ = 3/(3+1) = 0.75 quantile. That's the newsvendor critical ratio. The mean or median systematically under-supplies. In the backtest, q50 cost 25% more than q80.

### 2. "Why did q80 win if theory says q75?"
The fitted upper quantiles are slightly low, so the empirical minimum shifts up. I choose the operating point on data so that miscalibration shows up as a shifted quantile, rather than silent excess cost.

### 3. "How do you prevent leakage?"
Every feature is relative to the forecast origin and declares the newest hour it reads. Target-relative lags are allowed only if the lag is at least the horizon. Four tests: a contract check, perturbing everything after the origin and expecting identical features, a future-shock correlation test, and a code scan for random splits. The backtest also requires each training row's target to precede the first test origin.

### 4. "Why is a simple model tied with LightGBM?"
The per-zone weekly profile times a recent-level ratio captures most of the signal in this data, and it adapts to level shifts instantly. At small counts, Poisson noise caps everyone. LightGBM is shipped for its quantile family, not for accuracy. On real data with bigger counts and more structure, I'd expect the gap to widen.

### 5. "How does monitoring work?"
PSI and KS on demand-derived features, both pooled and per zone, plus rolling MASE. Calendar features are excluded because they "drift" by design. Pooled checks flagged only 2 of 16 features on a redistribution where half the zones rose and half fell. Per zone, all 8 were flagged.

### 6. "How do you decide to promote a new model?"
Champion vs challenger on the same backtest. The challenger must win by 2%. Aliases in the MLflow registry (`@staging`, `@production`). The first duel was a tie because there was no new data, which taught me to pace retraining to data arrival.

### 7. "Tell me about a bug."
The cost-weighted LightGBM trained a constant. LightGBM's sklearn and native APIs pass custom-objective arguments in different orders, and I had them swapped, which flipped the gradient. No error: every feature importance was zero. A test now asserts it correlates above 0.9 with quantile regression at τ*, since they're the same estimator.

### 8. "How do you know it works?"
Backtest cost against two baselines (−32% vs seasonal naive, −8% vs the hour-of-week mean), a CI regression gate against a committed baseline, dbt data tests, and leakage tests. But all on synthetic data. On real data, I'd re-run `make ingest && make run` and expect the numbers to move.

### 9. "How would you scale it to the whole city?"
All zones on a cloud warehouse with dbt, weather and event features, hierarchical or cross-zone models, shadow deployment, data versioning, multiple API replicas with shared metrics, and precomputed forecasts.

### 10. "What's MASE, and why two versions?"
Mean absolute scaled error: MAE divided by a naive forecast's MAE. The backtest scales by each fold's training data so folds compare to each other. The monitor uses a fixed reference so a rising MASE means real degradation. The two aren't comparable, and the code documents that.

### 11. "Why dbt and DuckDB?"
Tested, documented SQL transforms that run locally for free. Nine named cleaning rules with counts in a data-quality mart. Custom tests for hourly grid gaps and lag alignment.

### 12. "What would break first in production?"
Real-world variance (weather, events) the model can't see, then the single-node setup under city-wide load.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Synthetic data | "Is this real?" | "No. The TLC endpoint was blocked. The pipeline is real-data ready, and I'd expect different numbers." |
| Assumed 3:1 | "Why 3?" | "An assumption, set in config. It should be estimated from abandonment and idle-time economics." |
| Simple model tie | "Why ship LightGBM?" | "For calibrated quantiles and intervals, not accuracy. They're tied within noise." |
| No weather | "Big miss." | "Yes. It's likely the largest unexplained variance on real data." |
| Stray notebook | "What's Startup_Analysis.ipynb?" | "A leftover from another project in the early commits. It should be removed." |
