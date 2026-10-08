# Who to Target with Marketing: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like uplift or the Qini curve, are explained in [11-technical-terms.md](11-technical-terms.md).

This is a study, not a live service, so there's no web server or database. The tools are for analysis, testing and repeatability.

---

## The Language

### Python 3.10 to 3.12
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** almost every statistics and machine learning tool works with Python. The project is tested on three versions.

---

## Data and Statistics

### NumPy (version 2 or newer)
**What it is:** a Python tool for fast maths on long lists of numbers.

**Why it's used here:** it does the core maths, like the areas under Qini curves. The code uses a function only found in NumPy 2, so older versions won't work.

### pandas
**What it is:** a Python tool for working with tables of data, like a spreadsheet controlled by code.

**Why it's used here:** it loads and handles the 64,000-customer trial and all the result tables.

### SciPy
**What it is:** a Python tool for scientific maths and statistics.

**Why it's used here:** it provides the statistical tests, like the beats-random test and the power calculations for the next experiment.

---

## Building Models

### scikit-learn
**What it is:** Python's most common toolbox for everyday machine learning.

**Why it's used here:** it provides ordinary prediction models that the S, T and X recipes use as building blocks.

### LightGBM
**What it is:** a fast tool that builds many small yes/no flowcharts, each fixing the last one's mistakes.

**Why it's used here:** it's another building block for the S, T and X recipes. The blocks can be swapped to compare them.

### EconML
**What it is:** a Python library from Microsoft Research for estimating cause and effect with machine learning.

**Why it's used here:** it provides the causal forest, a specialised, well-tested uplift method. It's compared against the simpler hand-written recipes.

---

## Running the Study

### Command-line interface (CLI)
**What it is:** a way to run a program by typing commands.

**Why it's used here:** each stage is one command: validate, naive, hillstrom, experiment, or all of them. Every number in the memo traces back to a results file these commands create.

### Settings file (config.py)
**What it is:** one place in the code where all the business assumptions live.

**Why it's used here:** $0.10 per e-mail, a 30% margin and a 30% budget are all set here. Changing them in one spot updates the whole study.

---

## Quality Checks

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code works.

**Why it's used here:** 150 tests pass, all using made-up data. So they run anywhere, without downloading the real trial.

### ruff
**What it is:** a fast "linter": it flags mistakes and messy style in code.

**Why it's used here:** it keeps the code tidy and catches small errors.

### mypy (strict)
**What it is:** a tool that checks every value is used as the right kind, like not treating text as a number. "Strict" mode checks the most.

**Why it's used here:** in statistics code, a quiet type mix-up can give a wrong answer that still looks believable. mypy catches these early.

### GitHub Actions
**What it is:** a free service from GitHub that runs checks automatically when code changes.

**Why it's used here:** it runs the linter, type checks and tests on Python 3.10, 3.11 and 3.12 for every change.

---

## Data Sources

### Hillstrom e-mail trial
**What it is:** a public, randomised e-mail trial from 2008 with 64,000 customers.

**Why it's used here:** it's the real-world test. Randomisation makes a fair comparison possible.

### Criteo-UPLIFT (optional)
**What it is:** a much larger public uplift dataset from an advertising company.

**Why it's included:** the project can load it as an option. The main results don't use it.

---

## The Website

### HTML, CSS and JavaScript
**What it is:** the three languages every web page is made of: content, looks, and behaviour.

**Why it's used here:** the website is written in them directly, with no extra framework and no build step. So there's nothing to install, and very little that can break later. The charts are drawn by the project's own code too.

### GitHub Pages
**What it is:** free website hosting from GitHub, for sites made only of files.

**Why it's used here:** the site needs no server, because uploaded CSV files are analysed inside your browser. A workflow republishes it on every change, but only if the site's data file matches the latest results.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.10 to 3.12 | The language everything is written in |
| NumPy 2, pandas, SciPy | Maths, tables and statistical tests |
| scikit-learn + LightGBM | Building blocks for the uplift recipes |
| EconML | The causal forest method |
| CLI + settings file | Run each stage, keep assumptions in one place |
| pytest, ruff, mypy | Keep the code correct |
| GitHub Actions | Checks every change on 3 Python versions, and publishes the site |
| HTML, CSS, JavaScript + GitHub Pages | The free public website |
| Hillstrom trial | The real randomised data |
| Criteo-UPLIFT | Optional larger dataset |
