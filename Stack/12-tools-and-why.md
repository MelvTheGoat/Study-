# Reckon, Payment Matching: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software or service) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like webhooks or calibration, are explained in [11-technical-terms.md](11-technical-terms.md).

One choice stands out: the matching model is written by hand using only Python's built-in tools, with no machine learning library. With 18 features, that keeps every step readable.

---

## The Language

### Python 3.11
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** the whole service is written in it. The code is also fully "typed", meaning every value's kind is written down and checked, which catches mistakes early.

### Python's built-in tools (difflib and math)
**What it is:** tools that come with Python itself. `difflib` compares pieces of text, and `math` does maths.

**Why it's used here:** `difflib` helps score how alike two names are, and `math` powers the hand-written logistic regression model. No extra installs are needed, and every line can be read.

---

## The Web Service

### FastAPI
**What it is:** a Python tool for building web services and APIs (ways for programs to talk to each other).

**Why it's used here:** it receives Paystack's webhooks, serves the review pages, and offers a small API. It can reply straight away and do slow work afterwards, which stops Paystack from resending messages.

### Uvicorn
**What it is:** the program that actually runs a FastAPI service and answers requests.

**Why it's used here:** FastAPI needs something to run it, and Uvicorn is its standard partner.

### Jinja2
**What it is:** a tool for filling in HTML page templates with data, like a mail-merge for web pages.

**Why it's used here:** it builds the review queue and daily report pages.

### python-multipart
**What it is:** a small tool that lets FastAPI read form submissions from web pages.

**Why it's used here:** the review pages have approve and reject buttons, which send forms.

---

## Data and Settings

### SQLAlchemy 2
**What it is:** a Python tool for working with databases (organised stores of data in tables) without writing database commands by hand.

**Why it's used here:** it stores payments, invoices, decisions and the audit log. It uses SQLite (a single-file database) by default, and can switch to PostgreSQL (a bigger database) with one setting.

### Pydantic and pydantic-settings
**What it is:** Pydantic checks data has the right shape and types. pydantic-settings loads settings from a private `.env` file.

**Why it's used here:** it defines what a payment record looks like. The same definition also creates the fixed format the optional AI reader must answer in.

---

## Talking to Paystack

### httpx
**What it is:** a Python tool for sending requests to websites and APIs.

**Why it's used here:** it makes the "verify" call to Paystack for every payment. It's easy to fake in tests, so no real Paystack account is needed.

### Paystack
**What it is:** a Nigerian payments company.

**Why it's used here:** it's where payments come from. It sends webhooks, and its verify answer is the final word on each payment.

---

## Optional AI Reader

### A hosted AI provider's toolkit (optional)
**What it is:** the official Python toolkit for a hosted AI model service. It's an optional extra, not part of the normal install.

**Why it's used here:** it powers the last-resort reader for payment reports the patterns can't read. It's off by default, needs a key, and is only asked once per report. Its accuracy hasn't been measured yet.

---

## Quality Checks

### pytest (and pytest-cov)
**What it is:** pytest runs tests, which are small programs that check the code works. pytest-cov shows how much of the code the tests reach.

**Why it's used here:** 494 tests pass when the optional AI extra is installed. Without it, 8 tests for the AI reader fail, which the project's notes don't mention.

### mypy
**What it is:** a tool that checks every value is used as the right kind, like not treating text as a number.

**Why it's used here:** it runs in "strict" mode. For money code, catching a type mix-up before it runs is well worth it.

### ruff
**What it is:** a fast "linter": it flags mistakes and messy style in code.

**Why it's used here:** it keeps the code tidy and catches small errors.

### pre-commit
**What it is:** a tool that runs checks automatically every time you save a change to git.

**Why it's used here:** it makes sure the linter and other checks run before any change is saved.

---

## Running It

### Docker
**What it is:** a tool that packs an app and everything it needs into one box, called a container, that runs the same anywhere.

**Why it's used here:** Reckon is packaged as a container, ready to deploy.

### Google Cloud Run and Cloud Build
**What it is:** Cloud Run is a Google service that runs containers online. Cloud Build builds the container in the cloud.

**Why it's used here:** they're the planned home for Reckon. The deploy script refuses to ship any version that fails a quick smoke test. The project's notes say it hadn't been deployed yet.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.11 | The language, fully typed |
| difflib + math | Name matching and the hand-written model |
| FastAPI + Uvicorn | Receives webhooks and serves pages |
| Jinja2 | Builds the review and report pages |
| SQLAlchemy | Stores everything (SQLite or PostgreSQL) |
| Pydantic | Defines the payment record's shape |
| httpx | Confirms payments with Paystack |
| AI provider toolkit | Optional last-resort report reader |
| pytest, mypy, ruff, pre-commit | Keep the code correct and tidy |
| Docker + Cloud Run | Package and run it online |
