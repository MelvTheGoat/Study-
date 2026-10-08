# Job Hunt Helper: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like embeddings or upserts, are explained in [11-technical-terms.md](11-technical-terms.md).

Everything runs on your own computer, for free. No paid AI service is called by the code.

---

## The Language

### Python 3.10+
**What it is:** a popular programming language known for being easy to read. "3.10+" means version 3.10 or newer.

**Why it's used here:** the whole tool is written in Python. It's great for fetching data, working with text and running small AI models.

---

## Fetching Jobs

### requests
**What it is:** a simple Python tool for sending requests to websites and getting answers back.

**Why it's used here:** it fetches jobs from every ATS board and job board. The project wraps it in its own "polite" layer that waits between calls and retries failures.

### python-dotenv
**What it is:** a tool that reads secret settings, like API keys, from a private `.env` file.

**Why it's used here:** a few optional job sources need a free key. Keeping keys in `.env` means they never get saved into the shared code.

---

## Scoring Jobs

### sentence-transformers (all-MiniLM-L6-v2)
**What it is:** sentence-transformers is a Python tool for turning text into embeddings (lists of numbers that capture meaning). all-MiniLM-L6-v2 is a small, free model it can run, about 90 MB.

**Why it's used here:** it compares your CV and projects with each job by meaning. It runs on a normal computer with no GPU, and your CV never leaves your machine.

### NumPy
**What it is:** a Python tool for fast maths on lists of numbers.

**Why it's used here:** it does the maths for comparing embeddings (cosine similarity) and averaging chunks of text.

---

## Storing and Showing Results

### SQLite
**What it is:** a database (an organised store of data in tables) that lives in a single file, with no separate server.

**Why it's used here:** it stores every job, its label and score, your notes, and the saved embeddings. One file is simple and needs no setup.

### openpyxl
**What it is:** a Python tool for reading and writing Excel spreadsheets.

**Why it's used here:** it writes your `tracker.xlsx`, and also reads your edits back from it first. That way, notes you type in Excel are never lost.

### python-docx
**What it is:** a Python tool for creating Word documents.

**Why it's used here:** it builds a tailored CV for each job, as a Word file and a PDF, from one file of facts about you.

### PyYAML
**What it is:** a Python tool for reading YAML files, a simple, human-friendly settings format.

**Why it's used here:** almost every setting lives in YAML files: the 405 companies, score weights, skills, sources, sponsorship phrases and writing rules. You can tune the tool without touching the code.

---

## Running and Testing

### cron / Task Scheduler
**What it is:** built-in timers on Mac/Linux (cron) and Windows (Task Scheduler) that run a program at set times.

**Why it's used here:** they run the daily job search for you automatically, for free.

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code still works.

**Why it's used here:** 84 tests pass in a few seconds, using saved sample data, so no internet is needed. They cover merging copies, labels, the "never overwrite my notes" rule, spreadsheet edits and the letter checker.

---

## Outside the Code

### An AI chat assistant (for letters)
**What it is:** a chat-based AI that drafts text.

**Why it's used here:** the cover letters are drafted in a separate chat session, following writing rules kept in the project. The Python code doesn't write letters itself. It prepares the queue, then **checks** each letter for made-up numbers and banned phrases.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.10+ | The language everything is written in |
| requests | Fetches jobs from websites |
| python-dotenv | Keeps optional API keys private |
| sentence-transformers + MiniLM | Compares your CV with jobs by meaning |
| NumPy | Does the similarity maths |
| SQLite | Stores jobs, notes and saved embeddings |
| openpyxl | Writes and reads the Excel tracker |
| python-docx | Builds tailored CVs |
| PyYAML | Holds all the settings |
| cron / Task Scheduler | Runs the search daily |
| pytest | Checks the code works |
