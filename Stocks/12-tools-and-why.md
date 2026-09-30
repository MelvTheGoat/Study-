# Stock Question Agent: Tools and Why They Were Used

This file covers every tool (a ready-made piece of software) the project uses or plans to use. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like caching or backoff, are explained in [11-technical-terms.md](11-technical-terms.md).

The project is small on purpose. The code needs just **three** outside tools to run: Pydantic, PyYAML and httpx. Tools marked *(Planned)* aren't in use yet.

---

## The Language

### Python 3.10+
**What it is:** a popular programming language known for being easy to read. "3.10+" means version 3.10 or newer.

**Why it's used here:** almost every AI and data tool works with Python, so it's the natural choice for testing AI models.

---

## Settings

### Pydantic
**What it is:** a Python tool that checks data has the right shape and types, and gives clear errors when it doesn't.

**Why it's used here:** it checks every run's settings file strictly. Any unknown setting (like a misspelled "temparature") causes an error, instead of being silently ignored.

### PyYAML
**What it is:** a Python tool for reading YAML files. YAML is a simple, human-friendly format for settings, using indents and "name: value" lines.

**Why it's used here:** each run is described in a YAML file, like `configs/smoke.yaml`. It's easy for a person to read and edit.

---

## Talking to the Model

### httpx
**What it is:** a Python tool for sending requests over the web and getting answers back.

**Why it's used here:** it sends every question to the model server, with timeouts so a stuck server can't freeze a run. It also lets tests swap in a pretend connection, so tests never touch the real internet.

### vLLM *(Planned)*
**What it is:** a program that runs open AI models fast and lets other programs talk to them in OpenAI's message format.

**Why it's planned:** it's quick, free, and speaks the format the project's client already uses. So no client code needs to change when it's added.

### Qwen2.5-7B-Instruct-AWQ *(Planned)*
**What it is:** an open AI model from Alibaba. "7B" means about 7 billion internal numbers, and "AWQ" means it's been squeezed to use less memory.

**Why it's planned:** squeezed down, it's about 5 GB, so it fits on a free 16 GB GPU with room to spare. The settings file says the final choice of model will be made later.

### Kaggle *(Planned)*
**What it is:** a data science website that gives out free GPU time (special chips that run AI fast).

**Why it's planned:** it's free. The catch is a limit of about 30 hours a week, and sessions can be stopped. That's why the project caches answers and writes files safely.

---

## Testing and Checking

### pytest
**What it is:** a Python tool for running tests, which are small programs that check your code still works.

**Why it's used here:** 64 tests check the settings, the cache, the retries, the logging and the model client. They all pass in about 1.5 seconds.

### pytest-socket
**What it is:** an add-on for pytest that can block all internet access during tests.

**Why it's used here:** the tests are run a second time with the internet blocked. If any test quietly starts using the network, it fails straight away, not on the day a service happens to be down.

### ruff
**What it is:** a "linter": a tool that reads code and flags mistakes and messy style.

**Why it's used here:** it runs first in every automatic check, so small errors are caught early.

### GitHub Actions
**What it is:** a free service from GitHub (the website where the code is stored) that runs tasks automatically when code changes.

**Why it's used here:** it runs two checks on every change:

1. **Tests:** lint, then the tests, then the tests again with the internet blocked.
2. **Authorship:** fails if any file or commit message mentions an AI tool, to keep the project's authorship clean.

---

## Quick Summary

| Tool | Job in one line | Status |
|---|---|---|
| Python 3.10+ | The language everything is written in | Used |
| Pydantic | Checks the settings strictly | Used |
| PyYAML | Reads the settings files | Used |
| httpx | Sends questions to the model | Used |
| pytest + pytest-socket | Tests the code, also with the internet blocked | Used |
| ruff | Tidies and checks the code | Used |
| GitHub Actions | Runs the checks on every change | Used |
| vLLM | Runs the AI model fast | Planned |
| Qwen2.5-7B-Instruct-AWQ | The AI model to be tested | Planned |
| Kaggle | Free GPU time | Planned |
