# FPL AI Manager: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like shrinkage or the optimiser, are explained in [11-technical-terms.md](11-technical-terms.md).

One thing stands out: the back end (the part that does the work, out of sight) needs only **four** outside tools to run. That's on purpose. The points model is simple arithmetic, so heavy data tools aren't needed.

---

## The Language

### Python 3.11
**What it is:** a popular programming language known for being easy to read. 3.11 is the version.

**Why it's used here:** all the back-end work is in Python: collecting data, predicting points, picking teams and running jobs. It's the usual choice for data work.

---

## Collecting Data

### httpx
**What it is:** a Python tool for sending requests to websites and APIs and getting answers back.

**Why it's used here:** it fetches everything from the FPL API. It supports time limits, so a slow answer can't hang the whole run.

---

## Storing Data

### SQLite
**What it is:** a database (an organised store of data in tables) that lives in a single file, with no separate server.

**Why it's used here:** the whole season fits in a few megabytes. One file is easy to back up, zip, and store on a git branch between runs. The trade-off is that only one program can write to it at a time.

---

## Picking the Team

### PuLP and CBC
**What it is:** PuLP is a Python tool for writing "find the best choice under these rules" problems. CBC is the free solver that actually finds the answer.

**Why it's used here:** picking the best legal squad of 15 from about 650 players is exactly this kind of problem. According to the code, it normally solves in well under a second, with a 30-second safety limit.

---

## Serving the Results

### FastAPI
**What it is:** a Python tool for building an API, a service other programs can ask for data.

**Why it's used here:** it serves each team's picks, scores and player details as JSON (simple labelled data). It only reads data and never changes it, so it can't interfere with picking.

### Uvicorn
**What it is:** the program that actually runs a FastAPI service and answers requests.

**Why it's used here:** FastAPI needs something to run it. Uvicorn is the standard partner. Together with FastAPI, httpx and PuLP, these are the four outside tools the back end needs.

### React and Vite
**What it is:** React is a popular tool for building web pages out of reusable pieces. Vite is a tool that bundles those pieces into files a browser can load quickly.

**Why it's used here:** React draws the website: the half-pitch squad view, a tab for each team, player details and a season chart. Vite builds it into plain files for hosting.

---

## Running Itself

### GitHub Actions
**What it is:** a free service from GitHub (the website where the code is stored) that runs tasks for you, on a timetable or on request.

**Why it's used here:** it's the project's "server". At 7 and 37 minutes past every hour, it:

1. Loads the database from the `season-data` branch.
2. Runs the scheduler, which does whatever is due.
3. Deletes old predictions that are no longer needed.
4. Saves the database back, plus a 30-day backup copy.
5. Exports the data, builds the website and publishes it.

### GitHub Pages
**What it is:** free website hosting from GitHub for plain files.

**Why it's used here:** the site is read-only, so it doesn't need a live server. Every page's data is saved as a file ahead of time, and Pages simply hands them out.

### Docker and Render
**What it is:** Docker packs an app and everything it needs into one box, called a container, that runs the same anywhere. Render is a hosting service that can run that box all the time.

**Why it's used here:** they're an extra option, not the main setup. If you wanted an always-on server with its own storage, the same code can run this way.

---

## Keeping the Code Healthy

### pytest
**What it is:** a Python tool for running tests, which are small programs that check your code still works.

**Why it's used here:** a single wrong rule would quietly break every score. 610 tests pass in about 43 seconds, with no internet needed, because they use real recorded API answers.

### ruff
**What it is:** a "linter": a tool that reads your code and flags mistakes and messy style.

**Why it's used here:** it keeps the code tidy and catches small errors before they become bugs.

---

## Tools Deliberately *Not* Used

The project does **not** use pandas, NumPy or scikit-learn, the usual data and machine learning tools. The project's settings file says why: the predictions are simple arithmetic, not a model trained on data. Fewer tools means less to install and less to go wrong.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.11 | The language the back end is written in |
| httpx | Fetches data from the FPL API |
| SQLite | Stores the whole season in one file |
| PuLP + CBC | Finds the best legal squad |
| FastAPI + Uvicorn | Serves the results as data |
| React + Vite | Builds the website |
| GitHub Actions | Runs everything about every 30 minutes |
| GitHub Pages | Hosts the website for free |
| Docker + Render | An optional always-on setup |
| pytest + ruff | Test and tidy the code |
