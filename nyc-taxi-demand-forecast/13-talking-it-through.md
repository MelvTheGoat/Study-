# NYC Taxi Demand Forecast: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: all the results come from realistic made-up taxi data. The real New York data couldn't be downloaded when the project was built.*

---

## Okay, so what are we building?

Alright. Imagine you run taxi dispatch in New York. Every hour, in every part of the city, you have to decide how many drivers to send there. And that depends on a forecast: how many people will want a taxi?

Getting it wrong costs money both ways. Too few drivers, and riders wait and give up.

Too many, and drivers sit idle. But running short is worse. This project treats it as **3 times** as costly.

So here's our project: forecast taxi pickups for every zone, every hour, 24 hours ahead. And because running short is worse, the forecast should lean high on purpose. Plus, the system runs itself: it collects data, cleans it, tests itself honestly, watches for changes, and only swaps in a new model when it's clearly better.

## So what do we need?

1. **Settings** for every path and number.
2. **Made-up data** with the same layout as the real data, for testing.
3. **Safe downloading** of the real data.
4. **Cleaning**, with named rules, into an hourly table.
5. **Facts for the forecast** that never see the future.
6. **Models**, from simple guesses to LightGBM.
7. **A cost model** that leans high.
8. **An honest backtest.**
9. **Tracking**, and a library of saved models.
10. **Watching for changes**, and a fair way to replace the model.
11. **A forecast service.**
12. **Something that runs it all**, on a schedule.

## Step zero: set up the workshop

First, a `README.md`, the front page, and a `.gitignore`. `pyproject.toml` describes the project and its tools.

`Makefile` has a short command for every job. And the most important one is `make run`, which its notes call "the one command": the full pipeline, end to end, on made-up data.

The code lives in `src/`. Its `__init__.py` sums up the whole system in one line: ingest, transform, backtest, monitor for drift, retrain if needed and promote, then serve.

`src/config.py` holds every path and setting that more than one part uses. So the scheduler, the commands, the service and the tests all agree. And `src/logging_utils.py` has small helpers so every part writes its log messages the same way.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `pyproject.toml`: project details and tools.
> - `Makefile`: one short command per job, including "the one command".
> - `src/__init__.py`: marks the code folder as a package, with a summary.
> - `src/config.py`: every shared path and setting.
> - `src/logging_utils.py`: consistent log messages.

## Step one: made-up data first

Here's a choice that shapes the whole project. Before touching real data, we build a realistic fake.

`src/synthetic.py` makes taxi trip files with exactly the same layout as New York's real ones. Its notes call it "the backbone of the project's testability". It includes daily and weekly patterns, holidays, messy rows, and a planned shift in demand: half the zones go up about 38%, and half go down about 26%.

`scripts/make_synthetic.py` is the command to make it, with different sizes: a small one for automatic checks, and bigger ones for demos. Every test and every check uses this data, so no test ever needs the internet.

Tested in `tests/test_synthetic.py`. Its notes say the generator is the foundation every other test stands on, so its promises have to be checked.

> **📁 Files we just created**
> - `src/synthetic.py`: realistic fake taxi data, same layout as the real thing.
> - `scripts/make_synthetic.py`: the command to make it, in different sizes.
> - `tests/test_synthetic.py`: checks the fake data keeps its promises.

## Step two: downloading safely

`src/ingest.py` downloads the real monthly trip files from New York. It's "idempotent", which means running it again does nothing harmful. If a month is already there with a matching fingerprint, it's skipped.

It retries failed downloads, waiting longer each time, and it profiles missing values. And if the download fails completely, which it did when this was built, it falls back to the made-up data.

Tested in `tests/test_ingest.py`.

> **📁 Files we just created**
> - `src/ingest.py`: safe, repeatable downloads, falling back to made-up data.
> - `tests/test_ingest.py`: download tests.

## Step three: cleaning with dbt

Now cleaning. This is done with dbt, a tool for writing data-cleaning steps in SQL, the language of databases, and testing the results. It runs on DuckDB, a small database that runs inside your program.

The setup files: `dbt_project/dbt_project.yml` is dbt's main settings file. `dbt_project/profiles.yml` tells dbt to use a local DuckDB file, with no cloud and no cost. And `dbt_project/models/sources.yml` describes the raw data coming in.

Then three layers, each with one job.

**Staging**, the first layer. `stg_yellow_trips.sql` renames columns, sets the right types, and removes copies, with no business rules yet. `stg_taxi_zones.sql` loads the list of zones, dropping placeholder "unknown" zones. `staging.yml` documents and tests that layer.

## The middle layer

**Intermediate**, the middle layer. `int_trips_flagged.sql` applies every cleaning rule as a *named* yes-or-no flag, rather than quietly deleting rows. There are 9 rules, like "distance must be above zero" and "speed must be possible". `int_trips_cleaned.sql` keeps only the rows that pass every rule.

`int_analysis_window.sql` sets the date range, because the raw data has a few records with impossible dates. And `int_hour_spine.sql` builds a dense grid: one row for every zone and every hour, even with zero pickups. As its notes say, this is what makes the "same hour last week" facts correct. `intermediate.yml` documents and tests it all.

## The final layer

**Marts**, the final layer. `fct_hourly_zone_demand.sql` is what we forecast: pickups per zone per hour, on that full grid. `mart_demand_features.sql` adds the facts: calendar, holidays, past values and rolling averages. `dim_zones.sql` describes the zones that actually have demand.

And `mart_data_quality.sql` has one row per cleaning rule, saying how many rows it rejected. That makes the cleaning visible, not a mystery. `marts.yml` documents and tests these.

US holidays come from `dbt_project/seeds/dim_us_holidays.csv`. A seed is a small fixed table. It's made once by `scripts/make_holiday_seed.py`, so dbt doesn't need Python while it builds.

## dbt's own tests

There are custom tests too:

- `tests/assert_demand_matches_cleaned_trips.sql`: total demand must equal the number of cleaned trips, so no rows are lost or doubled.
- `tests/assert_lag_features_align.sql`: "same hour last week" really points at the right hour. It's a data-side guard against leakage.
- `tests/generic/test_no_gaps_in_hourly_grid.sql`: every zone's hours are exactly one hour apart.
- `tests/generic/test_accepted_range.sql`: values stay in a sensible range.
- `tests/generic/test_unique_combination_of_columns.sql`: no duplicate zone-hours.

`src/transform.py` runs dbt the same way every time, for the scheduler, the commands and the tests. Tested in `tests/test_transform.py`.

> **📁 Files we just created**
> - `dbt_project/dbt_project.yml`, `dbt_project/profiles.yml`, `dbt_project/models/sources.yml`: dbt's setup.
> - `dbt_project/models/staging/stg_yellow_trips.sql`, `stg_taxi_zones.sql`, `staging.yml`: the first layer.
> - `dbt_project/models/intermediate/int_trips_flagged.sql`, `int_trips_cleaned.sql`, `int_analysis_window.sql`, `int_hour_spine.sql`, `intermediate.yml`: the middle layer, with the 9 named rules and the full grid.
> - `dbt_project/models/marts/fct_hourly_zone_demand.sql`, `mart_demand_features.sql`, `dim_zones.sql`, `mart_data_quality.sql`, `marts.yml`: the final tables.
> - `dbt_project/seeds/dim_us_holidays.csv` and `scripts/make_holiday_seed.py`: the holiday table and how it's made.
> - `dbt_project/tests/assert_demand_matches_cleaned_trips.sql`, `assert_lag_features_align.sql`: custom data tests.
> - `dbt_project/tests/generic/test_no_gaps_in_hourly_grid.sql`, `test_accepted_range.sql`, `test_unique_combination_of_columns.sql`: reusable data tests.
> - `src/transform.py`: runs dbt the same way every time.
> - `tests/test_transform.py`: transform tests.

Let's pause. We have made-up and real data, safe downloads, and clean data in a full hour-by-zone table, with every rule named and tested. Now the forecasting.

## Step four: facts that never see the future

`src/features.py` builds the facts for each forecast. Why not just use the facts table from dbt? Because of one problem: the forecast horizon.

Say we're forecasting 24 hours ahead. "Pickups one hour ago" isn't known yet at that point. So every fact here is measured from the moment the forecast is made, and declares how recent its newest data is. If a fact would use data from after that moment, the code refuses to build it.

It also makes sure the facts are built the same way for training and for the live service.

And `tests/test_leakage.py`, whose notes say it's the test "this project would be worthless without". A leaky backtest reports a score it can't repeat in real life, and it does so silently. There's also `tests/test_features.py`.

> **📁 Files we just created**
> - `src/features.py`: facts measured from the forecast moment, refusing any leaks.
> - `tests/test_leakage.py`: the tests the project would be worthless without.
> - `tests/test_features.py`: other fact tests, including training and live parity.

## Step five: models and costs

`src/models.py` holds the forecasters. The simple guesses come first: "same as this hour last week" and "the average for this hour of the week". Then a per-zone weekly pattern model, and LightGBM, which can forecast many "safety levels" at once. All take the same input and give the same kind of output.

Now `src/cost.py`, and its notes give the framing: unmet demand is three times worse than idle supply. Here's the function that picks the right safety level:

```python
def optimal_quantile(
    cost_under: float = DEFAULT_COST_UNDER, cost_over: float = DEFAULT_COST_OVER
) -> float:
```

Its notes say it plainly: for 3 to 1, it's 0.75. So aim for the 75th quantile, a number demand stays under 75% of the time. But the project also checks this on real results, and on the test data, 0.80 did slightly better.

Tests: `tests/test_models.py` and `tests/test_cost.py`. The cost tests check the maths itself, like making sure 3 to 1 really is 3 to 1.

> **📁 Files we just created**
> - `src/models.py`: simple guesses, a weekly pattern model, and LightGBM.
> - `src/cost.py`: the 3-to-1 cost, and how to pick the safety level.
> - `tests/test_models.py` and `tests/test_cost.py`: model and cost tests.

## Step six: the honest backtest

`src/backtest.py`, and its notes call it "the core of the project". It tests on the past as if it were live. Pick a moment, train on data before it, forecast the next week, then move forward and repeat, 6 times.

And it's strict about honesty. No random splits, anywhere. And a training row is only allowed if its *outcome* happened before the test period starts, not just its start time. That subtle rule stops a sneaky kind of leak.

There's even a code scan that blocks random shuffling functions from being used at all. Tested in `tests/test_backtest.py`.

> **📁 Files we just created**
> - `src/backtest.py`: the honest backtest, the core of the project.
> - `tests/test_backtest.py`: backtest tests.

## Step seven: tracking, and a library of models

`src/tracking.py` uses MLflow, a tool that records every training run and keeps a library of saved models. Each model gets a label, like "production" or "staging". The service always uses whatever's labelled "production".

Let's circle back. We have clean data, honest facts, models, a cost-based safety level, an honest backtest and a model library. The forecasting brain is done. Now the system needs to look after itself.

> **📁 Files we just created**
> - `src/tracking.py`: records every run, and labels models in a library.

## Step eight: watching, and replacing fairly

`src/monitoring.py` watches for changes. And its rule is: retrain because of evidence, never because of the calendar.

It checks whether recent demand looks different from normal, using two standard measures, PSI and a KS test. And it checks city-wide *and* zone by zone. That matters: when some zones rose and others fell, the city-wide check mostly missed it, because it nearly balanced out. The zone check caught every zone.

Then `src/retrain.py`, with one rule: a retrained model is a **challenger, not a replacement**. It must beat the current champion by at least 2% on the same test. Otherwise the champion stays, and the reason is written down.

`src/regression_gate.py` is another safety net. The project saves a baseline of its backtest numbers in `baselines/backtest_baseline.json`. `scripts/ci_regression_check.py` re-runs a small backtest on every change, and fails the build if results get more than 10% worse.

Tests: `tests/test_monitoring.py` and `tests/test_promotion.py`.

> **📁 Files we just created**
> - `src/monitoring.py`: watches for changes, city-wide and zone by zone.
> - `src/retrain.py`: a challenger must win by 2% to replace the champion.
> - `src/regression_gate.py`: compares results with a saved baseline.
> - `baselines/backtest_baseline.json`: the saved baseline.
> - `scripts/ci_regression_check.py`: fails the build if results get 10% worse.
> - `tests/test_monitoring.py` and `tests/test_promotion.py`: monitoring and promotion tests.

## Step nine: serving and running it all

`src/serve.py` is the forecast service, built with FastAPI. You ask for a zone and how far ahead, and it returns the recommended number for each hour, plus a range from low to high. There are also addresses for a health check, live measurements and model details. Tested in `tests/test_serve.py`.

`src/flows.py` runs the whole thing with Prefect, a scheduling tool: ingest, clean, backtest, watch, and retrain only if needed. It runs a check every hour and the full pipeline every day. Tested in `tests/test_flows.py`.

Then packaging. `Dockerfile` packs it into a box called a container, and `.dockerignore` says what to leave out. `docker-compose.yml` starts the pipeline, the forecast service and the MLflow screen, sharing one storage space because they genuinely share data.

`.github/workflows/ci.yml` checks every change. Shared test setup is in `tests/conftest.py`, all on made-up data.

> **📁 Files we just created**
> - `src/serve.py`: the forecast service.
> - `src/flows.py`: runs the whole pipeline on a schedule.
> - `tests/test_serve.py` and `tests/test_flows.py`: service and schedule tests.
> - `Dockerfile` and `.dockerignore`: pack it into a box.
> - `docker-compose.yml`: starts the pipeline, service and MLflow screen together.
> - `.github/workflows/ci.yml`: checks every change.
> - `tests/conftest.py`: shared test setup.

## Step ten: writing it all down

- `ARCHITECTURE.md`: why each piece is shaped the way it is, and what was rejected.
- `RUNBOOK.md`: how to run it, retrain it, roll back a bad model, and fix common problems.
- `EXPLAINER.md`: written for reading before an interview, explaining every decision.
- `CODE_WALKTHROUGH.md`: the project's own build-order walkthrough.

And one stray file: `Startup_Analysis.ipynb`. It's a notebook from a different project, left over in the main folder. It isn't part of this system.

> **📁 Files we just created**
> - `ARCHITECTURE.md`: the design reasons.
> - `RUNBOOK.md`: how to operate the system.
> - `EXPLAINER.md`: every decision explained, for interviews.
> - `CODE_WALKTHROUGH.md`: the project's own walkthrough.
> - `Startup_Analysis.ipynb`: a stray notebook from another project.

## So, how's it doing?

On the made-up data, with 8 zones and 6 test weeks:

- **LightGBM's MASE was 0.83**, so it beat the "same as last week" guess.
- **Total cost was about 32% lower** than that simple guess.
- **The simple weekly pattern model tied with LightGBM.** LightGBM is used because it gives a full range of safety levels.
- **The safety level matters a lot.** Using the middle (50%) level instead of 80% cost 25% more.
- **167 tests pass.**

## What's still missing?

- **Only made-up data** has been used so far.
- **No weather or events**, which drive real demand spikes.
- **The 3-to-1 cost is an assumption.**
- **Each zone is forecast separately.**
- **A new model replaces the old in one step**, with no trial period.
- **A stray notebook** from another project sits in the main folder.

## Let's put it all together

So let's look at it in one breath.

We set up the workshop, with **one command** to run everything. We built **realistic fake data** first, so everything is testable offline. We **downloaded safely**, falling back to the fake data. We **cleaned with dbt** in three layers, with **9 named rules**, a **full hour-by-zone grid**, and tests for every promise.

Then **facts that never see the future**, measured from the forecast moment. **Models** from simple guesses to LightGBM, and a **cost model** that leans high. An **honest backtest**, and a **model library** with labels.

Finally, a system that **looks after itself**: watching **zone by zone**, replacing the model only when a **challenger wins by 2%**, a **regression gate**, a **forecast service**, and a **scheduler**.

Notice how it links. The full grid from step three is what makes the "same hour last week" facts in step four correct. The 3-to-1 cost from step five decides which safety level gets served in step nine. And the made-up data from step one is what lets every single test run without the internet.

That's the project. Lean high on purpose, test honestly, and only change the model when there's proof.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
