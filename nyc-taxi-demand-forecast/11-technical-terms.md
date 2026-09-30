# NYC Taxi Demand Forecast: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow the data's journey: collecting it, cleaning it, making features, forecasting, testing, watching for changes, and running it all.

---

## 1. Collecting the Data

### TLC trip data
**What it means:** New York's Taxi and Limousine Commission publishes a file of every taxi trip each month.

**Why it's needed here:** it's the real data source the project is built for. The download was blocked when it was built, so no real data has been used yet.

### Parquet
**What it means:** a compact file format for large tables of data.

**Why it's needed here:** the monthly trip files come as parquet.

### Synthetic data
**What it means:** realistic, made-up data with the same layout as the real thing.

**Why it's needed here:** it lets everything run and be tested without the real download. It includes daily and weekly patterns, holidays, messy rows, and a planned shift where half the zones go up about 38% and half go down about 26%.

### Idempotent
**What it means:** doing something twice has the same effect as doing it once.

**Why it's needed here:** the daily download can safely run again and again without duplicating data.

### Checksum manifest
**What it means:** a checksum is a fingerprint of a file. A manifest is a list of those fingerprints.

**Why it's needed here:** if a file's fingerprint hasn't changed, it's skipped, which saves time and avoids reprocessing.

### Retry with backoff
**What it means:** trying a failed download again, waiting longer each time.

**Why it's needed here:** downloads fail for temporary reasons. Waiting a bit longer each time usually gets them through.

---

## 2. Cleaning and Shaping

### Data warehouse (DuckDB)
**What it means:** a database designed for analysing large amounts of data. DuckDB is a small one that runs inside your program, with no separate server.

**Why it's needed here:** all the cleaning and table-building happens in DuckDB. It's fast, free and simple.

### SQL
**What it means:** the standard language for asking questions of databases and reshaping tables.

**Why it's needed here:** the cleaning steps are written in SQL, through dbt.

### dbt (data build tool)
**What it means:** a tool for writing data-cleaning steps as SQL files, running them in order, and testing the results.

**Why it's needed here:** it turns raw trips into a clean table in documented, tested steps.

### Staging, intermediate and mart
**What it means:** three layers of tables. Staging is raw data with the right types. Intermediate is cleaned data. Marts are the final tables ready to use.

**Why it's needed here:** each layer does one job, so problems are easy to find.

### Data quality rules
**What it means:** named checks that reject bad rows, like "distance must be above zero" or "speed must be possible".

**Why it's needed here:** there are 9 rules. A separate report counts how many rows each rule removed, so cleaning is never a mystery.

### Dense grid
**What it means:** a table with a row for every zone and every hour, even when nothing happened.

**Why it's needed here:** a missing hour and a zero-pickup hour look the same otherwise. A dbt test checks there are no gaps.

---

## 3. Features

### Feature
**What it means:** one fact turned into a number that a model can learn from, like "pickups at this hour last week".

**Why it's needed here:** models learn from features. Most here are past demand, calendar facts and holidays.

### Lag and rolling average
**What it means:** a lag is a past value, like "demand 168 hours ago" (one week). A rolling average is the average over a recent window.

**Why it's needed here:** taxi demand repeats every day and every week, so past values are strong clues.

### Forecast origin and horizon
**What it means:** the origin is the moment a forecast is made. The horizon is how far ahead it looks, here 24 hours.

**Why it's needed here:** features must only use data from before the origin. For a forecast 24 hours ahead, "demand one hour ago" isn't known yet.

### Data leakage
**What it means:** when a model accidentally uses information it wouldn't have in real life.

**Why it's needed here:** it makes results look far better than they really are. Every feature declares its newest data, and the code refuses any that would leak. A code scan also blocks random shuffling of the data.

### Calendar features
**What it means:** facts about the date, like hour of day, day of week and holidays.

**Why it's needed here:** demand depends strongly on time. They're left out of drift checks, because they naturally change all the time.

---

## 4. Forecasting

### Seasonal naive
**What it means:** the simplest forecast: "same as this hour last week".

**Why it's needed here:** it's the baseline. A useful model must beat it.

### Seasonal profile
**What it means:** each zone's typical weekly pattern, scaled by how busy it's been recently.

**Why it's needed here:** it's a simple but strong model. It tied with LightGBM on accuracy.

### LightGBM
**What it means:** a tool that builds many small yes/no flowcharts, each fixing the last one's mistakes.

**Why it's needed here:** it can forecast many safety levels (quantiles) directly, which the dispatcher's decision needs.

### Quantile
**What it means:** a value that demand stays below a given share of the time. The 80th quantile is exceeded only 20% of the time.

**Why it's needed here:** the dispatcher needs a safe number, not the average. The project forecasts from the 10th to the 90th quantile.

### Asymmetric cost
**What it means:** when one kind of mistake costs more than the other.

**Why it's needed here:** running short of drivers is assumed to cost 3 times as much as having too many. That's why forecasts lean high.

### Newsvendor rule
**What it means:** a classic rule for choosing how much to stock when running out and having extra cost different amounts. The best level is: cost of running short ÷ (both costs added).

**Why it's needed here:** it says 3 ÷ (3 + 1) = 0.75, so the 75th quantile. On the data, the 80th did slightly better, so the choice is checked on data.

### Pinball loss
**What it means:** a scoring method for quantile forecasts that punishes being too low and too high by different amounts.

**Why it's needed here:** it's how LightGBM learns each quantile, and it matches the asymmetric cost idea exactly.

### Marginal quantile
**What it means:** a quantile for one hour on its own, not for several hours together.

**Why it's needed here:** it's a known limit. You can't add up the 90th quantiles of several hours to get the 90th quantile of the total.

---

## 5. Testing

### Backtest (rolling origin)
**What it means:** testing on the past as if it were live. You pick a moment, train on data before it, forecast after it, then move the moment forward and repeat.

**Why it's needed here:** it's the honest way to test forecasts. There are 6 test rounds (folds), each a week long.

### Expanding window
**What it means:** each new test round trains on all data up to that point, so training data grows over time.

**Why it's needed here:** it copies real life, where you keep all your history.

### MAE and MASE
**What it means:** MAE is the average size of a miss. MASE compares that to the simple "same as last week" guess. Below 1 means better than the simple guess.

**Why it's needed here:** MASE makes results easy to judge. LightGBM scored 0.83.

### Regression gate
**What it means:** an automatic test that fails if results get worse than a saved baseline.

**Why it's needed here:** every code change is checked. If results get more than 10% worse, the change is blocked.

---

## 6. Watching and Updating

### Drift
**What it means:** when data patterns change over time.

**Why it's needed here:** the model learned old patterns. If demand shifts, forecasts get worse.

### PSI and KS test
**What it means:** two standard ways of measuring how much a set of numbers has shifted compared to normal.

**Why it's needed here:** they're the drift alarms. They run both city-wide and zone by zone.

### Per-zone monitoring
**What it means:** checking each zone separately, instead of only the city as a whole.

**Why it's needed here:** when some zones rose and others fell, the city-wide check flagged only 2 of 16 cases, while per-zone checks caught all 8 zones.

### Champion and challenger
**What it means:** the champion is the current model. A challenger is a new model trying to replace it.

**Why it's needed here:** when drift is found, a challenger is trained. It must beat the champion by at least 2%, or the champion stays.

### Model registry and aliases
**What it means:** a registry is a library of saved models. Aliases are labels, like "production" or "staging".

**Why it's needed here:** the API always uses whatever is labelled "production". Promoting a model just moves the label.

### Shadow deployment
**What it means:** running a new model alongside the old one for a while, without using its answers, to check it in real conditions.

**Why it's needed here:** the project doesn't do this yet. Promotion happens in one step, which is a known limit.

---

## 7. Running It

### Orchestration
**What it means:** automatically running steps in the right order, at the right times.

**Why it's needed here:** a drift check runs every hour, and the full pipeline every day, with no one pressing buttons.

### API
**What it means:** a way for programs to ask for something, here "give me the forecast".

**Why it's needed here:** the dispatcher's system asks for forecasts. It gets the recommended number, plus a range from the 10th to the 90th quantile.
