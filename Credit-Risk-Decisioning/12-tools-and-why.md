# Loan Approval Decisions: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software or service) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like calibration or reason codes, are explained in [11-technical-terms.md](11-technical-terms.md).

---

## The Language

### Python 3.11
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** almost every data and machine learning tool works with Python. The code is also "typed", meaning each value's kind is written down and checked.

---

## Working with Data

### NumPy and pandas
**What it is:** NumPy does fast maths on long lists of numbers. pandas works with tables of data, like a spreadsheet controlled by code.

**Why it's used here:** they generate the 30,000 synthetic applications, split them by month, and handle every table of results.

---

## Building Models

### scikit-learn
**What it is:** Python's most common toolbox for everyday machine learning (programs that learn patterns from data).

**Why it's used here:** it provides logistic regression, the scorecard's underlying model, both calibration methods, and many of the measures, like AUC.

### LightGBM
**What it is:** a fast tool that builds many small yes/no flowcharts, each fixing the last one's mistakes.

**Why it's used here:** it's the best of the three models. It catches combinations of risk factors that simpler models miss.

### statsmodels
**What it is:** a Python tool for classic statistics.

**Why it's used here:** it runs the Heckman method, one of the four ways tested for guessing what declined applicants would have done.

A warning: the project allows any statsmodels version from 0.14 up. Version 0.15 removed an option the code uses, so on a fresh install 4 tests fail. Limiting it to below 0.15 fixes this.

### SHAP
**What it is:** a tool that explains a single prediction by showing how much each feature pushed it up or down.

**Why it's used here:** it explains the LightGBM model's decisions. The live decline reasons use scorecard points instead, because those can be reproduced exactly later.

---

## Serving Decisions

### FastAPI and Uvicorn
**What it is:** FastAPI is a Python tool for building APIs (ways for programs to ask for things). Uvicorn is the program that runs it.

**Why it's used here:** they serve the decision service, with three addresses: one for decisions, one to check it's healthy, and one to describe the model.

### Pydantic
**What it is:** a tool that checks data has the right shape, types and ranges.

**Why it's used here:** it checks every application before scoring, like "income must be a positive number". Bad input is rejected with a clear error.

### Streamlit
**What it is:** a Python tool for building simple web pages with little code.

**Why it's used here:** it provides a demo form. You fill in an application and see the decision, the chance of default, the expected loss and the reasons. Its default values are outside the range the model learned from, so the demo scores unusual applicants unless you change them.

---

## Packaging and Shipping

### Docker (multi-stage)
**What it is:** Docker packs an app and everything it needs into one box, called a container. "Multi-stage" means one stage builds things, and a smaller final stage only keeps what's needed to run.

**Why it's used here:** the build stage trains the model inside the box, so the model and its calibration always come from one run. The final box doesn't run with full admin rights, which is safer.

### GitHub Actions
**What it is:** a free service from GitHub that runs checks automatically when code changes.

**Why it's used here:** it runs the linter, type checks, tests and the calibration gate. It then builds the Docker box, checks it starts, and uploads it.

### AWS ECR (Elastic Container Registry)
**What it is:** Amazon's online storage for Docker boxes.

**Why it's used here:** finished boxes are uploaded here, ready to run on Amazon's cloud. One catch: the upload step builds a fresh box, which retrains the model, rather than reusing the box that passed the checks.

---

## Quality Checks

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code works.

**Why it's used here:** 227 tests pass (with statsmodels below 0.15). They include the calibration gate and a check that the API and batch scoring give identical results.

### ruff and mypy
**What it is:** ruff is a "linter" that flags mistakes and messy style. mypy checks every value is used as the right kind.

**Why it's used here:** in money-related code, catching small mistakes before running matters a lot.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.11 | The language, fully typed |
| NumPy + pandas | Make and handle the data |
| scikit-learn | Logistic regression, calibration, measures |
| LightGBM | The best-performing model |
| statsmodels | One bias-correction method (pin below 0.15) |
| SHAP | Explains individual predictions |
| FastAPI + Uvicorn | The decision service |
| Pydantic | Checks every application's fields |
| Streamlit | The demo form |
| Docker | Packages the app with its model |
| GitHub Actions | Tests, checks and ships every change |
| AWS ECR | Stores the finished boxes |
| pytest, ruff, mypy | Keep the code correct and tidy |
