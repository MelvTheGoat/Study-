# NYC Taxi Demand Forecast: System Design for Beginners

This project predicts how many taxi pickups each New York City zone will have, every hour, for the next 24 hours. A dispatcher can then send drivers to where riders will be. It runs itself: it collects data, cleans it, tests itself honestly, watches for changes, and only swaps in a new model when it's clearly better. All the results so far come from realistic made-up data, because the real data couldn't be downloaded when it was built.

## Key Terms

- **Forecast**: a prediction of a future number, like "12 pickups in Midtown at 6 p.m. tomorrow".
- **Zone**: one area of the city. Taxi data is grouped by zone.
- **Model**: a program that learns patterns from past data to make a guess.
- **Quantile**: a "safety level" for a forecast. The 75th quantile is a number demand stays below 75% of the time.
- **Backtest**: testing a forecaster on the past, as if it were live, without letting it see the future.
- **Drift**: when patterns change over time, like demand moving to different zones.
- **Champion and challenger**: the current model, and a new one trying to replace it.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The forecast drives a decision: how many drivers to send. Too few drivers means riders wait and give up, which is worse than a driver waiting. So the forecast should lean high on purpose.

**Step 2: Figure out the data.** New York publishes every taxi trip each month. The project reads it, and falls back to a realistic generator with the same layout when the download fails.

**Step 3: Sketch the main parts.** Data is collected, cleaned into an hourly table, turned into features, and used to train and test models. A service then shares the forecasts.

**Step 4: Walk through one forecast.** Follow one zone's forecast for 6 p.m. tomorrow, from raw trips to the number the dispatcher sees.

**Step 5: Decide how to know it works.** Measure errors against simple guesses, and the total cost of being wrong.

**Step 6: Plan for problems.** Future data can leak into training, demand can shift, and a new model might only look better by luck. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Forecast **every hour**, for the next **24 hours**, for each zone.
- Treat running short as **3 times** as bad as having too many drivers.
- Check for changes **every hour**, and run the full pipeline **every day**.
- Only replace the model if the new one is at least **2%** better.

Why lean high? Step by step:

1. Running short costs 3, and having too many costs 1.
2. The best safety level is 3 ÷ (3 + 1) = **0.75**.
3. So aim for the 75th quantile: a number demand stays under 75% of the time.
4. On the test data, 80% worked slightly better, so the project checks this on data rather than trusting the maths alone.

### The Big Picture (Step 3)

```
   [Taxi trip data (or made-up data)]
                 |
                 v
   [Ingest: download safely]
                 |
                 v
   [Clean + build hourly zone table]
                 |
                 v
   [Features: no peeking ahead]
                 |
                 v
   [Train + backtest models] ---> [Model Registry]
                                        |
                                        v
   [Drift Monitor] --breach--> retrain   [Forecast API]
```

Step by step:

1. Each day, Ingest downloads new monthly trip files, skipping any it already has.
2. The cleaning step removes bad rows using 9 named rules, and builds a table with one row per zone per hour.
3. Features are made, like "pickups at this hour last week", using only data from before the forecast is made.
4. Several models are trained and tested on the past, week by week.
5. The best one is saved in the Model Registry, labelled "production".
6. Every hour, the Drift Monitor checks for changes. If it spots one, it trains a challenger and compares.
7. The Forecast API gives the dispatcher the recommended number, plus a range.

### The Main Parts (Step 3)

**Ingest.** It downloads files safely: it retries on failure, and uses a fingerprint to skip files it already has. It's like a postman who doesn't deliver the same letter twice.

**Cleaning and Hourly Table.** It uses dbt (a tool for writing and testing data-cleaning steps in SQL, the language of databases). Bad rows, like trips with zero distance or impossible speeds, are removed and counted by rule. Every zone gets a row for every hour, even with zero pickups. It's like a register where every pupil is marked present or absent.

**Features, with No Peeking.** Each feature must say how recent its data is. If a feature would use data from after the forecast is made, the code refuses to build it. It's like an exam where you can't see tomorrow's newspaper.

**Models.** Simple guesses, like "same as this hour last week", are the baselines to beat. A per-zone weekly pattern model and LightGBM (a tool that builds many small yes/no flowcharts) are the main contenders. LightGBM gives a whole range of safety levels, which is why it's the one used.

**Drift Monitor.** It checks whether recent demand looks different from normal, both for the whole city and zone by zone. The city-wide check missed a shift where some zones rose and others fell. The per-zone check caught it, like checking every room for a leak, not just the water bill.

**Champion vs Challenger.** A new model must beat the current one by at least 2% on the same test. Otherwise the old one stays, and the reason is written down. It's like a boxing title: the challenger must clearly win.

How the backtest folds grow over time is a detail worth learning later (advanced - skip for now).

### How We Know It's Working (Step 5)

These come from the made-up data: 8 zones, 6 test weeks.

- **MASE**: error compared with the simple "same as last week" guess. Below 1 means better. LightGBM scored **0.83**.
- **Total cost**: LightGBM cost about **32% less** than the simple guess.
- **Safety level matters**: using the 50th quantile instead of the 80th cost **25% more**.
- **Tests**: 167 automatic checks pass.

### What Can Go Wrong (Step 6)

- **Peeking at the future.** Features declare their newest data, and training rows only use outcomes from before the test period.
- **Demand moves between zones.** The per-zone drift check catches what the city-wide check misses.
- **A lucky challenger.** The 2% margin stops the model being swapped for random noise.
- **Made-up data.** Real demand has weather, events and bigger spikes, so real results may be worse.

## Quick Recap

- Running short costs 3 times more, so forecast the 75th to 80th quantile, not the average.
- Clean data into a full hour-by-zone table, with named rules.
- Features must never use data from after the forecast is made.
- Watch for drift zone by zone, not just city-wide.
- A new model must clearly win before it replaces the old one.
