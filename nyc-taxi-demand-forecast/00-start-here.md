# NYC Taxi Demand Forecast: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: all the results come from realistic **made-up** taxi data. The real New York data couldn't be downloaded when the project was built. The system is ready for real data, but hasn't used it yet.

## 1. The Problem

A taxi dispatcher has to decide, every hour and in every part of the city, how many drivers to send there. That decision depends on a forecast: how many people will want a taxi?

Getting it wrong costs money both ways. Too few drivers, and riders wait and give up. Too many, and drivers sit idle. But running short is worse: this project treats it as **3 times** as costly as having too many.

## 2. The Big Idea

Because running short is worse, the best forecast isn't the average. It should lean high on purpose.

The project forecasts a "safety level" instead of an average. A 75% safety level is a number that demand stays under 75% of the time. A simple formula says 75% is the right level for a 3-to-1 cost. On the test data, 80% worked slightly better, so the project checks this on data rather than trusting the formula alone.

The second big idea: the system runs itself. It collects data, cleans it, tests itself, watches for changes, and only swaps in a new model when it's clearly better.

## 3. How It Works, Step by Step

**Step 1: Download the trip data.** New York publishes every taxi trip each month. The project downloads new months safely, retrying on failure and skipping files it already has. If the download fails, it uses realistic made-up data with the same layout.

**Step 2: Clean the data.** Bad trips are removed using 9 named rules, like "distance must be above zero" and "speed must be possible". A report counts how many trips each rule removed.

**Step 3: Build an hourly table.** It makes a table with one row for every zone (area of the city) and every hour, even hours with zero pickups.

**Step 4: Make facts for the forecast.** Facts include "pickups at this hour last week", the day of the week, and holidays. Every fact must only use data from before the forecast is made.

**Step 5: Train and test several models.** Simple guesses like "same as this hour last week" are the baselines to beat. The main contenders are a per-zone weekly pattern model and LightGBM, a tool that builds many small yes/no flowcharts.

**Step 6: Test honestly on the past.** It pretends to be at a point in the past, trains on data before it, and forecasts the next week. Then it moves forward a week and repeats, 6 times. This is called a backtest.

**Step 7: Save the best model.** The chosen model is saved in a model library and labelled "production".

**Step 8: Watch for changes every hour.** A monitor checks whether recent demand looks different from normal, both city-wide and zone by zone.

**Step 9: Challenge the champion.** If the monitor spots a change, a new model (the challenger) is trained. It replaces the current one (the champion) only if it's at least 2% better. Otherwise the old one stays, and the reason is written down.

**Step 10: Share the forecasts.** A web service gives the dispatcher the recommended number for each zone and hour, plus a range from low to high.

## 4. The Clever Parts

**Leaning high on purpose.** Forecasting a safety level instead of an average fits the real costs. Using the middle (50%) level instead of the 80% level cost 25% more.

**No peeking at the future.** Every fact must say how recent its data is. If a fact would use data from after the forecast is made, the code refuses to build it. A scan of the code also blocks random shuffling of the data, which could leak future information.

**Checking zone by zone.** In the made-up data, some zones got busier while others got quieter. City-wide, it almost balanced out, so the city-wide check mostly missed it. The zone-by-zone check caught every zone.

**A fair challenge.** The 2% margin stops the model being swapped because of random luck.

**One command runs it all.** The whole system starts with a single command.

## 5. The Important Words

- **Forecast**: a prediction of a future number.
- **Zone**: one area of the city.
- **Horizon**: how far ahead the forecast looks, here 24 hours.
- **Quantile (safety level)**: a number that demand stays under a given share of the time.
- **Asymmetric cost**: when one kind of mistake costs more than the other.
- **Backtest**: testing on the past as if it were live.
- **Leakage**: accidentally using future information.
- **Drift**: when patterns change over time.
- **Champion and challenger**: the current model, and a new one trying to replace it.
- **MASE**: error compared with a simple "same as last week" guess. Below 1 means better.

## 6. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **DuckDB**: a small database where the cleaning happens.
- **dbt**: writes and tests the cleaning steps in SQL, the language of databases.
- **LightGBM**: the main forecasting model.
- **MLflow**: records every model and labels the production one.
- **Prefect**: runs the hourly and daily jobs automatically.
- **Evidently**: helps with reports about changing data.
- **FastAPI**: serves the forecasts.
- **Docker Compose**: starts everything with one command.
- **pytest**: runs 167 automatic checks.

## 7. How Good Is It?

These come from the made-up data: 8 zones, 6 test weeks.

- **LightGBM's MASE was 0.83**, so it beat the simple "same as last week" guess.
- **Total cost was about 32% lower** than the simple guess.
- **The simple per-zone pattern model tied with LightGBM.** LightGBM is used because it gives a full range of safety levels, not because it's more accurate.
- **The safety level matters as much as the model.** Choosing the wrong level cost 25% more.

## 8. What's Weak or Missing

- Only made-up data has been used so far.
- Real demand has weather, events and bigger spikes, which aren't included.
- The 3-to-1 cost is an assumption, not measured.
- Each zone is forecast separately, so neighbouring zones don't share information.
- A new model replaces the old one in one step, with no trial period running side by side.
- There's a stray notebook file from a different project in the main folder.

## 9. What This Project Shows You Can Do

- Turn a business cost into the right kind of forecast.
- Build a clean, tested data pipeline.
- Test forecasts honestly, with no peeking at the future.
- Run a system that watches itself and updates safely.
- Be honest about the limits of made-up data.

## 10. Ten Things to Remember

1. It forecasts hourly taxi pickups per zone, 24 hours ahead.
2. Running short is treated as 3 times worse than too many drivers.
3. So it forecasts a high safety level (75% to 80%), not the average.
4. Data is cleaned with 9 named rules into a full hour-by-zone table.
5. Facts must never use data from after the forecast is made.
6. It tests itself on 6 past weeks, moving forward one week at a time.
7. Drift is checked zone by zone, not just city-wide.
8. A challenger must be at least 2% better to replace the champion.
9. LightGBM cut cost by about 32% against the simple guess.
10. All results so far come from made-up data.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
