# Stock Research: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software or service) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like stale data or peers, are explained in [11-technical-terms.md](11-technical-terms.md).

One choice stands out: every service here is free at this size. The setup guide says so plainly, and lists the paid options separately for the owner to decide on.

---

## The Language

### Python 3.11+
**What it is:** a popular programming language known for being easy to read. "3.11+" means version 3.11 or newer.

**Why it's used here:** it has good tools for data, files and web pages, and the earlier version of the project was already in Python.

---

## Getting the Data

### httpx
**What it is:** a Python tool for sending requests over the web and getting answers back.

**Why it's used here:** it downloads accounts from the SEC and prices from Twelve Data. It makes timeouts easy, and lets tests swap in stand-ins so they never touch the real internet.

### SEC EDGAR
**What it is:** the US regulator's free public system of company filings.

**Why it's used here:** it gives every US company's accounts, plus industry lists used to find peers. It's free and public domain, as long as each request includes a contact email and stays under 10 a second.

### Twelve Data (free plan)
**What it is:** a share price service with a free plan of 800 requests a day and 8 a minute.

**Why it's used here:** it gives US prices, dividends and splits. The project fetches only new days to stay inside the limit, and guards against three traps in its answers.

### pypdf
**What it is:** a Python tool for reading text out of PDF files.

**Why it's used here:** it reads the main figures out of Nigerian companies' results PDFs. It can't read scanned PDFs, which have no text in them.

### PyYAML
**What it is:** a Python tool for reading YAML files, a simple, human-friendly format of "name: value" lines.

**Why it's used here:** the watchlist and the Nigerian company files are YAML, so they're easy to edit right on GitHub.

---

## Storing the Data

### Parquet (with pyarrow)
**What it is:** a compact file format for tables. pyarrow is the Python tool that reads and writes it.

**Why it's used here:** prices, dividends and splits are kept as plain Parquet files, which are small and easy to move around.

### DuckDB
**What it is:** a database that runs inside your program, with no server to set up.

**Why it's used here:** it lets the code ask questions of the Parquet files using SQL, the standard database language.

---

## Building the Website

### Jinja2
**What it is:** a Python tool for filling in page templates with data.

**Why it's used here:** every company page uses the same layout, filled in with its own figures, sentences and charts. The output is plain HTML.

### HTML, CSS and a little JavaScript
**What it is:** the three languages of web pages: content, looks and behaviour.

**Why it's used here:** the pages are plain HTML with light and dark themes. One small script adds chart hover, search and the compare picker, and every page still works without it.

---

## Running It and Publishing It

### GitHub Actions
**What it is:** a free service from GitHub (the website where the code is stored) that runs tasks automatically, on a timetable or when files change.

**Why it's used here:** it runs four jobs:

1. **Nightly:** fetch, build and publish the site every weekday night, keeping the data in its cache.
2. **NGX inbox:** read any uploaded results PDF and write the draft figures.
3. **SEC samples:** fetch real SEC files for the tests, because the SEC refuses the development machine.
4. **Tests:** lint, then the tests, then the tests again with the internet blocked.

### Cloudflare Pages
**What it is:** a free service that hosts websites made of plain files.

**Why it's used here:** the nightly job uploads the built pages there. There's nothing to run, so it costs nothing.

### Cloudflare Access
**What it is:** a login screen Cloudflare puts in front of a site. It's free for up to 50 users.

**Why it's used here:** the price data is for personal use, so only the owner's email is let in, with a one-time code. The nightly job won't publish until this is set up.

### ntfy
**What it is:** a free service and phone app for sending yourself notifications.

**Why it's used here:** it sends one message a night when something changes for a watched company, like a new report or a failed check.

---

## Testing and Checking

### pytest
**What it is:** a Python tool for running tests, which are small programs that check your code still works.

**Why it's used here:** 436 tests pass, using real SEC filings and made-up prices. Comments show the sum behind each expected number.

### pytest-socket
**What it is:** an add-on for pytest that can block all internet access during tests.

**Why it's used here:** the tests run a second time with the internet blocked, so a test that secretly calls a real service fails straight away.

### ruff
**What it is:** a "linter": a tool that reads code and flags mistakes and messy style.

**Why it's used here:** it runs first in every automatic check, so small errors are caught early.

---

## Quick Summary

| Tool | Job in one line | Status |
|---|---|---|
| Python 3.11+ | The language everything is written in | Used |
| httpx | Downloads from the SEC and Twelve Data | Used |
| SEC EDGAR | US company accounts and industry lists | Used |
| Twelve Data | US prices, dividends and splits | Used (free plan) |
| pypdf | Reads Nigerian results PDFs | Used |
| PyYAML | Reads the watchlist and Nigerian files | Used |
| Parquet + DuckDB | Store prices and query them | Used |
| Jinja2 | Builds the web pages | Used |
| GitHub Actions | Nightly build, PDF inbox, samples, tests | Used |
| Cloudflare Pages + Access | Private, free hosting | Set-up steps written |
| ntfy | Phone alerts | Optional |
| pytest, pytest-socket, ruff | Check the code | Used |
