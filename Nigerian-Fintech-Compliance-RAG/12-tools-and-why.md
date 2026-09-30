# Nigerian Fintech Rules Q&A: Tools and Why They Were Used

This file covers every tool and service the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like embeddings or rank fusion, are explained in [11-technical-terms.md](11-technical-terms.md).

The big theme is **staying small**. The app must fit in 2 GB of memory on a free server, so heavy tools were swapped for light ones wherever possible.

---

## The Language

### Python 3.10 to 3.12
**What it is:** a popular programming language known for being easy to read.

**Why it's used here:** the whole app is written in it. The automatic checks test it on three Python versions to make sure it works on each.

---

## The App

### Streamlit
**What it is:** a Python tool for building simple web pages with just a few lines of code.

**Why it's used here:** it's the fastest way to a usable page. It shows the question box, the answer, the source passages and any warnings.

---

## Preparing Documents

### pypdf
**What it is:** a Python tool for reading text out of PDF files.

**Why it's used here:** official regulations come as PDFs. It's written in pure Python, so it's easy to install anywhere.

---

## Searching

### ONNX Runtime and tokenizers
**What it is:** ONNX Runtime is a light tool for running AI models saved in the ONNX format. `tokenizers` splits text into the small pieces the model reads.

**Why it's used here:** together they run the MiniLM embedding model without the usual heavy toolkit (PyTorch). That saves hundreds of megabytes of memory and gives the same results.

### all-MiniLM-L6-v2
**What it is:** a small, free model that turns text into embeddings (lists of numbers that capture meaning).

**Why it's used here:** it powers the "meaning search". It's small enough to run on a free server.

### NumPy
**What it is:** a Python tool for fast maths on lists of numbers.

**Why it's used here:** it stores all 100 embeddings in one table and does the meaning search in a single calculation. At this size, no special search database is needed.

### Home-made BM25
**What it is:** the project's own short version of BM25, a standard keyword-ranking formula.

**Why it's used here:** it's tiny and needs no extra install. Keyword search matters a lot for rules full of exact terms.

---

## Writing Answers

### httpx
**What it is:** a Python tool for sending requests to web services.

**Why it's used here:** it sends the prompt to whichever AI service is chosen.

### Groq, Gemini or OpenAI
**What it is:** companies that offer AI models as online services. Groq has a free tier.

**Why it's used here:** one of them writes the final answer. You choose with a setting, and a free Groq key is enough. There's also a built-in "stub" that just quotes passages, for free testing.

---

## Quality Checks

### pytest
**What it is:** a Python tool for running tests, which are small programs that check the code works.

**Why it's used here:** 110 tests pass in about 1 second, with no internet or keys needed.

### ruff
**What it is:** a fast "linter": it flags mistakes and messy style in code.

**Why it's used here:** it keeps the code tidy and catches small errors early.

### mypy
**What it is:** a tool that checks every value is used as the right kind, like not treating text as a number.

**Why it's used here:** it catches mix-ups before the code runs.

---

## Running It

### Docker
**What it is:** a tool that packs an app and everything it needs into one box, called a container, that runs the same anywhere.

**Why it's used here:** it packages the app, including the ready-made search index, for deployment.

### Google Cloud Run
**What it is:** a Google service that runs containers online, and can scale down to zero when nobody's using it.

**Why it's used here:** it's the planned home for the app, with steps in the project's deploy guide. Peak memory use of about 225 MB fits easily in the 2 GB the project aims for.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.10 to 3.12 | The language everything is written in |
| Streamlit | The web page |
| pypdf | Reads text from PDFs |
| ONNX Runtime + tokenizers | Runs the meaning model with little memory |
| all-MiniLM-L6-v2 | Turns text into embeddings |
| NumPy | Does the meaning search |
| Home-made BM25 | Does the keyword search |
| httpx | Sends prompts to the AI service |
| Groq / Gemini / OpenAI | Writes the final answer |
| pytest, ruff, mypy | Keep the code correct and tidy |
| Docker + Cloud Run | Package and run it online |
