# Forecast the 75th percentile, not the average: a self-operating taxi demand system

Repo: https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

## Why I built it

Most forecasting projects end with a notebook and an error score. But a forecast is only useful as the input to a decision. For a taxi dispatcher, that decision is: **how many drivers should I position in each zone, each hour?**

I wanted to build the whole thing, not just the model: scheduled ingestion, tested transformations, honest backtesting, drift monitoring, safe retraining and an API, all running from one command. And I wanted every design choice to follow from the business problem.

One honest note first: **every number in this post comes from synthetic data.** The NYC TLC download endpoint was blocked in the environment where I built this, so ingestion falls back to a seeded generator with the same schema. The pipeline is ready for real data. The figures aren't real-world yet.

## The one assumption that shapes everything

Being wrong in the two directions costs different amounts:

- **Under-forecast:** a rider opens the app, waits, gives up. You lose the fare, driver utilisation, and maybe the rider.
- **Over-forecast:** a driver is sent to a zone that didn't need them. Fuel and idle time. Annoying but recoverable.

I priced the first at three times the second:

```
cost(y, ŷ) = 3 · max(y − ŷ, 0)  +  1 · max(ŷ − y, 0)
```

That's the pinball (quantile) loss. Minimising it doesn't give you the average. It gives you the **quantile at 3 / (3 + 1) = 0.75**. So a model trained to predict the mean and shipped at its median is optimising the wrong thing *and* shipping the wrong number.

That drives the design: models fit a *family* of quantiles (q10 to q90), the quantile to act on is **chosen from backtest cost**, and results are ranked by money, not error.

## How it works

### Ingestion and transformation

Monthly TLC parquet files are downloaded with retries and a checksum manifest, so re-running skips anything unchanged. dbt on DuckDB then builds the data in layers: staging (typed and deduplicated), intermediate (**nine named cleaning rules**, each counted in a data-quality mart), and marts (a dense hour × zone grid with calendar, holiday and lag features). Custom dbt tests check there are no gaps in the hourly grid, that lags line up, and that demand matches the cleaned trips.

### Features that can't see the future

This was the part that mattered most. The mart has `lag_1h`, the demand one hour earlier. That's fine for a one-hour-ahead forecast, and **leakage** for a 24-hour one. At 09:00 today you're asked about 09:00 tomorrow, and 08:00 tomorrow hasn't happened yet.

So every model feature is defined relative to the **forecast origin**, the last hour actually observed, and each one declares the newest hour it reads:

| Feature | Reads | Allowed? |
|---|---|---|
| `OriginLag(k)` | `y[origin − k]` | always |
| `OriginRolling(w)` | the window ending at origin | always |
| `TargetLag(k)` | `y[origin + h − k]` | only if `k ≥ max horizon` |
| `CalendarFeature` | the clock | always |

The feature builder refuses to produce a matrix whose declarations reach past the origin. Four tests attack this from different angles: a contract check, **overwriting everything after the origin with noise and checking no feature changed**, a correlation check on a series whose future is independent of its past, and a scan of the code for random splits like `train_test_split` or `shuffle=True`. (That scan's first run flagged one of my own docstrings, so it now scans code, not prose.)

The backtest adds one more barrier: a training row is only admitted if its **target** hour is before the first test origin. Filtering on the origin alone would let a training row's target land inside the test window.

### Models

- **Seasonal naive** and **hour-of-week mean** as baselines.
- **`SeasonalProfileZone`**: each zone's hour-of-week shape from all history, multiplied by its level over the last 168 hours. It's one pass over a group-by, and it tracks a level shift instantly.
- **LightGBM quantile family**: one model per quantile.

### Choosing the number to act on

The cost module computes total cost at every quantile on the backtest and picks the cheapest:

```python
def optimal_quantile(
    cost_under: float = DEFAULT_COST_UNDER, cost_over: float = DEFAULT_COST_OVER
) -> float:
    """Quantile that minimises the asymmetric cost in expectation.

    This is the newsvendor critical ratio; for 3:1 it is 0.75.
```

In theory the answer is 0.75. In the backtest it was **0.80**, a 1.2% difference. That isn't a discovery. It's a calibration warning: the fitted upper quantiles run slightly low, so the cost minimum drifts up. Selecting on data means miscalibration shows up as a shifted operating point rather than silent extra cost.

## Results (synthetic)

Rolling-origin backtest: 6 folds × 7-day test windows, 24-hour horizon, 8 zones, about 32,000 zone-hours per model.

| Model | MASE | Total cost | vs seasonal naive |
|---|---|---|---|
| LightGBM global (shipped at q0.80) | 0.825 | **104,568** | −32.1% |
| Seasonal profile per zone | **0.818** | 104,782 | −32.0% |
| Cost-weighted LightGBM | 0.938 | 108,648 | −29.5% |
| Hour-of-week mean | 0.863 | 113,594 | −26.2% |
| Seasonal naive | 1.042 | 154,010 | — |

Three things stand out:

1. **The operating point is worth as much as the model.** The same LightGBM shipped at the median costs **25% more** than at q80. The whole best-to-worst model gap is 32%.
2. **A simple per-zone profile ties gradient boosting.** It wins on MAE, RMSE and MASE and loses on cost by 0.2%, well inside fold noise. LightGBM is shipped for its calibrated quantile family, not for better accuracy.
3. **Beat the strong baseline, not the weak one.** Seasonal naive loses by 32%, but the hour-of-week mean only by 8%.

## What didn't work

- **Pooled drift detection missed the drift entirely.** The synthetic data moves half the zones up about 38% and half down about 26%. Pooled checks flagged 2 of 16 features. Per-zone checks flagged **8 of 8** zones. Aggregation hid the event, so drift is now checked both ways.
- **Calendar features made the monitor cry wolf.** `month_of_year` showed PSI of 5–9 on a healthy pipeline, because the calendar moves. Only demand-based features are monitored now.
- **The cost-weighted LightGBM silently trained a constant.** LightGBM's sklearn and native APIs pass custom-objective arguments in different orders. Getting it backwards flipped the gradient, and the model still fitted and scored, with every feature importance at zero. A test now checks it correlates above 0.9 with quantile regression at 0.75, since they're the same estimator written two ways. Once fixed, it's still 3% worse and can't produce intervals, so the quantile family stays.
- **The first champion/challenger duel was a model against itself.** No new data had arrived, so both scored 17,064. The 2% margin correctly refused to promote, but it showed retraining must be paced to data arrival, not the clock.

## Keeping it healthy on its own

Prefect runs an hourly monitor and a daily pipeline. The monitor checks PSI and KS (pooled and per zone) plus rolling MASE. On a breach, a challenger is trained and has to beat the production model by **2%** on the same backtest before MLflow's `@production` alias moves. Otherwise the system holds and logs why. CI compares backtest metrics with a committed baseline. The API returns the `operating` forecast (the cost-optimal quantile) plus q10/q50/q75/q90, with a note that shipping q50 would under-supply.

## What I learned

- **Start from the decision.** The cost function set the target, the models and the metric.
- **Leakage in forecasting is about the origin**, not just the train/test split.
- **Simple models deserve a fair fight.** One tied here.
- **Aggregates hide things.** Check drift where it actually happens.
- **Silent failures need tests that encode what "correct" must look like.**

## What's next

Run on real TLC data, add weather and events, model neighbouring zones together, add a shadow period before promotion, estimate the 3:1 ratio from real economics, and version the data.
