# nyc-taxi-demand-forecast: LinkedIn Post

*About 165 words. Copy from the line below.*

---

A demand forecast shouldn't predict the average. Mine predicts the 75th percentile, and here's why.

I built a system that forecasts hourly NYC taxi pickups per zone, 24 hours ahead, for dispatch. An under-forecast (a rider waits and leaves) costs about 3× an over-forecast (a driver idles). With that cost, the best forecast is the 3/(3+1) = 75th percentile, not the mean.

In the backtest, the same model shipped at the median cost 25% more than at the cost-optimal quantile. That's almost as much as the whole gap between the best model and a naive baseline (32%).

Other lessons:
- A simple per-zone weekly profile tied gradient boosting
- Drift checks on pooled data missed a shift where half the zones rose and half fell. Per-zone checks caught all 8.
- A challenger model must beat the champion by 2% before it replaces it

It runs end to end from one command: dbt + DuckDB, LightGBM quantiles, MLflow, Prefect, FastAPI.

Caveat: the numbers are from synthetic data. The real TLC download was blocked where I built it.

https://github.com/MelvTheGoat/nyc-taxi-demand-forecast

#Forecasting #MLOps #DataEngineering #MachineLearning
