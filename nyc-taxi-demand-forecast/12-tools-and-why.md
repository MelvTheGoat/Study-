# NYC Taxi Demand Forecast: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like quantiles or drift, are explained in [11-technical-terms.md](11-technical-terms.md).

Everything here is free and runs on one computer. The whole system starts with one command.

---

## The Language

### Python
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** it's the standard for data and forecasting work, and ties all the other tools together.

### pandas, NumPy and SciPy
**What it is:** pandas works with tables of data. NumPy does fast maths on lists of numbers. SciPy adds statistics, like the KS test.

**Why it's used here:** they handle the forecast tables, the maths, and one of the two drift checks.

---

## Collecting Data

### requests and tenacity
**What it is:** requests downloads files from the web. tenacity automatically retries things that fail, waiting longer each time.

**Why it's used here:** together they download the monthly taxi trip files reliably, even when the connection is shaky.

---

## Cleaning Data

### DuckDB
**What it is:** a small, fast database for analysing data. It runs inside your program, with no separate server.

**Why it's used here:** it's the project's "warehouse", where all the cleaning and table-building happens. It's free and needs no setup.

### dbt (with dbt-duckdb)
**What it is:** a tool for writing data-cleaning steps as SQL files (SQL is the language of databases), running them in order, and testing the results. dbt-duckdb connects it to DuckDB.

**Why it's used here:** it turns raw trips into a clean hour-by-zone table using 9 named rules. Its tests check things like "no missing hours" and "total demand equals cleaned trips".

---

## Forecasting

### LightGBM
**What it is:** a fast tool that builds many small yes/no flowcharts, each fixing the last one's mistakes. It can forecast quantiles (safety levels) directly.

**Why it's used here:** it's the main model. One version is trained for each safety level, from the 10th to the 90th quantile, so the dispatcher gets a full range.

---

## Tracking and Monitoring

### MLflow
**What it is:** a tool that records every training run (settings, results, model files) and keeps a library of saved models with labels.

**Why it's used here:** every model is traceable. The label "production" marks the model the API uses. Promoting a challenger just moves the label.

### Evidently
**What it is:** a tool for making reports about data drift, meaning how much data has changed over time.

**Why it's used here:** it supports the drift reports, alongside the project's own PSI and KS checks.

---

## Running It Automatically

### Prefect 3
**What it is:** a tool for scheduling and running workflows (chains of steps) written in Python.

**Why it's used here:** it runs the drift check every hour and the full pipeline every day, automatically.

### FastAPI
**What it is:** a Python tool for building APIs (ways for programs to ask for things).

**Why it's used here:** it serves the forecasts. There are addresses for forecasts, a health check, live measurements and model details.

### Docker Compose
**What it is:** Docker packs an app into a box, called a container, that runs the same anywhere. Compose starts several boxes together from one settings file.

**Why it's used here:** it starts the whole system with one command.

### Makefile
**What it is:** a file of named shortcuts for commands, run with `make`.

**Why it's used here:** common jobs, like running the pipeline or the tests, are each one short command.

---

## Quality Checks

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code works.

**Why it's used here:** 167 tests pass, taking about 3 minutes. They include checks that no feature can see the future.

### ruff and mypy
**What it is:** ruff is a "linter" that flags mistakes and messy style. mypy checks every value is used as the right kind.

**Why it's used here:** they catch small mistakes before they cause problems.

### GitHub Actions
**What it is:** a free service from GitHub that runs checks automatically when code changes.

**Why it's used here:** it runs the tests and the regression gate. If forecasts get more than 10% worse than the saved baseline, the change is blocked.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python, pandas, NumPy, SciPy | The language, tables and maths |
| requests + tenacity | Download trip files reliably |
| DuckDB | The local data warehouse |
| dbt | Cleans and tests the data in SQL |
| LightGBM | Forecasts every safety level |
| MLflow | Records runs and labels the production model |
| Evidently | Drift reports |
| Prefect 3 | Runs hourly and daily jobs |
| FastAPI | Serves the forecasts |
| Docker Compose + Makefile | One-command setup and shortcuts |
| pytest, ruff, mypy | Keep the code correct |
| GitHub Actions | Checks every change automatically |
