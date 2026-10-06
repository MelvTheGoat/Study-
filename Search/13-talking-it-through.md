# Job Hunt Helper: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. Picture this. You're looking for machine learning and AI jobs, and you're applying from Nigeria. Every day, you'd have to check hundreds of company career pages and job websites.

And then the hard part: which of these jobs can you actually get?

Is it remote and open worldwide? Is it in Nigeria or Africa? Will the company help with a visa? Most job posts don't say clearly.

So here's our project: a personal job-search helper that runs on your own computer. Every day it collects jobs, removes copies, works out which ones are open to you, and scores how well each fits your CV. It keeps everything in a tracker.

And one rule: it **never applies for you**. A person reads every job and every letter, and applies themselves. The helper just makes sure you read the best, reachable jobs first.

## So what do we need?

1. **Your profile**: your CV, your projects, your details, written down once.
2. **Settings**, so you can tune it without touching code.
3. **A list of companies** to check.
4. **Polite fetching**, so websites don't block us.
5. **Readers** for each job website.
6. **Removing copies**, since jobs appear in many places.
7. **A relevance filter**, to drop non-ML jobs.
8. **A way to tell if a job is open to you**, with evidence.
9. **A fit score** against your CV.
10. **A database and a tracker** that never lose your notes.
11. **Help with applications**: a letter queue, a letter checker, and tailored CVs.

## Step zero: set up the workshop

First, a `README.md`, the front page. A `.gitignore`. And `requirements.txt`, the list of tools to install.

There's also `scripts/setup.sh`, a one-time setup for a new computer. One nice touch: it installs the small, computer-only version of PyTorch, which is much smaller than the default.

`.env.example` is a template for optional API keys. A few job websites, like Adzuna and Reed, need a free key. You copy this file to `.env`, fill in any keys you have, and the private copy is never saved to git.

The code lives in a folder called `hunt`. `hunt/__init__.py` marks it as a package. `hunt/config.py` knows where everything lives: the settings folder, the data folder, your profile, the letter queue. And `hunt.py` at the top is the starting point you run, like `python hunt.py run`.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `requirements.txt`: the tools to install.
> - `scripts/setup.sh`: one-time setup on a new computer.
> - `.env.example`: a template for optional API keys.
> - `hunt/__init__.py`: marks the code folder as a package.
> - `hunt/config.py`: knows where every folder lives.
> - `hunt.py`: the starting point you run.

## Step one: write down who you are

Before we look for jobs, we write down who we're looking for jobs *for*. That's the `profile` folder.

`profile/profile.yaml` holds your details: name, email, GitHub, LinkedIn, location in Lagos, time zone, and availability. `profile/cv.md` is your CV, written in plain text. `profile/projects.md` holds extra GitHub projects that aren't on the CV, with every fact taken from each project's own README.

And `profile/cv_data.yaml` is your CV as structured facts. The tailored CVs get built from it later.

Why do this first? Because the scorer compares jobs against your CV, and the letter checker compares letters against it. Your profile is the measuring stick for everything.

> **📁 Files we just created**
> - `profile/profile.yaml`: your personal details.
> - `profile/cv.md`: your CV in plain text.
> - `profile/projects.md`: extra projects, with facts from each README.
> - `profile/cv_data.yaml`: your CV as structured facts.

## Step two: the settings

Now the settings. They all live in YAML files in a `config` folder, so you can tune the tool without touching any code.

- `config/companies.yaml`: the 405 company job boards to check, each with its company name, which job system it uses, and its country.
- `config/sources.yaml`: which job websites to use, and how gently to hit them, like the minimum seconds between calls.
- `config/scoring.yaml`: how the fit score is built. The weights, the seniority multipliers, and the title patterns.
- `config/skills.yaml`: the skills to look for in jobs and in your CV. A skill in the job but not in your CV becomes a "gap".
- `config/sponsorship.yaml`: the phrases that signal visa sponsorship, or the lack of it, like "unable to sponsor".
- `config/writing.yaml`: the rules for checking letters, like which dash characters to flag.

And `config/companies_removed.yaml` is where broken company boards go. Each one has a note saying why it was removed. Fix it, and you move it back.

> **📁 Files we just created**
> - `config/companies.yaml`: the 405 company boards.
> - `config/companies_removed.yaml`: boards that stopped working, with reasons.
> - `config/sources.yaml`: which job websites, and how gently to use them.
> - `config/scoring.yaml`: how the fit score is built.
> - `config/skills.yaml`: skills to look for.
> - `config/sponsorship.yaml`: phrases that signal visa sponsorship.
> - `config/writing.yaml`: the letter-checking rules.

## Step three: fetch jobs politely

Now we fetch jobs. And the first rule is: be polite.

`hunt/http.py` is the polite fetcher. It says clearly who it is, waits a minimum time between calls to the same website, retries failures with growing waits, and gives up after a time limit. Being polite keeps you from being blocked, and some websites ask for it.

Then we need one shape for every job, wherever it came from. That's `hunt/models.py`: one job, with its source, company, title, location and so on. Every source gets turned into this shape.

`hunt/text.py` has small text helpers, like turning a job post's web code into plain readable text.

Now the readers. `hunt/sources/ats.py` reads company job boards on 9 job systems, like Greenhouse, Lever and Ashby.

These are the original, fullest job posts. `hunt/sources/boards.py` reads job websites, like RemoteOK, Remotive and the "Who is hiring?" thread on Hacker News. Optional sources with keys only run if you've added a key.

And `hunt/sources/__init__.py` saves results from busy job websites for a few hours. Why? Because Remotive, for example, asks for no more than four calls a day.

Then `hunt/verify.py`, the health check. It tests every company board. If one is dead, it tries other job systems for that company. If nothing works, it moves the company to `companies_removed.yaml` with a note.

Tests: `tests/test_http.py` checks the polite fetching. `tests/test_sources.py` checks the readers, using saved real answers in `tests/fixtures/`: `greenhouse.json`, `lever.json`, `ashby.json` and `remotive.json`.

> **📁 Files we just created**
> - `hunt/http.py`: the polite fetcher.
> - `hunt/models.py`: one shape for every job.
> - `hunt/text.py`: turns web code into plain text.
> - `hunt/sources/ats.py`: reads company boards on 9 job systems.
> - `hunt/sources/boards.py`: reads job websites.
> - `hunt/sources/__init__.py`: saves busy websites' results for a few hours.
> - `hunt/verify.py`: checks every board, and repairs or removes broken ones.
> - `tests/test_http.py` and `tests/test_sources.py`: fetching tests.
> - `tests/fixtures/greenhouse.json`, `lever.json`, `ashby.json`, `remotive.json`: saved real answers for tests.

## Step four: remove copies, keep what's relevant

The same job often appears on several websites. `hunt/dedupe.py` merges copies. Two posts with the same apply link, or the same company, title and location, count as one. It keeps the one with the longest description, because the company's own post usually beats a short summary.

Then we drop jobs that aren't about data, machine learning or AI. That's a function called `is_relevant`, in the scorer file. Here's the real start of it:

```python
def is_relevant(title, description, cfg):
    """Step 2: keep jobs about data, ML or AI. Drops only clear misses."""
```

It drops clear misses by title, and keeps clear matches. For vague titles like "engineer", it counts machine learning words in the description, and needs enough of them.

Tested in `tests/test_dedupe.py`.

> **📁 Files we just created**
> - `hunt/dedupe.py`: merges copies of the same job.
> - `tests/test_dedupe.py`: tests the copy-merging.

## Step five: can you actually get this job?

Now the important bit. Is this job open to you?

First, `hunt/countries.py` finds countries in messy locations like "London, UK" or "Remote - EMEA". It knows country names, other names, and big cities, like Lagos, Abuja and Ikeja for Nigeria.

Then `hunt/labels.py` gives every job a label: remote and open, in Nigeria, in Africa, sponsors visas, probably sponsors, unknown, or restricted. And here's the key design choice: it saves **the exact sentence** it used. Something like "must be based in the US". Rules can be wrong, so you can always check its reasoning.

It also knows that ECOWAS countries, the West African group including Nigeria, need no visa for Nigerians.

But what if a job post says nothing about visas? That's where official lists help. `hunt/registers.py` downloads the UK and Netherlands lists of companies allowed to sponsor visas, refreshed weekly.

And `hunt/h1b.py` checks US visa filings. A company with at least 5 filings, one in the last 3 years, counts as a likely sponsor. Those results are saved for 30 days.

Then `hunt/reach.py` decides what to hide. Restricted jobs, jobs abroad with no sign of sponsorship, senior roles, 5+ years required, students only, or Master's or PhD required. They're hidden from your list but kept in the database. That way they don't come back as "new" tomorrow.

Tests: `tests/test_labels.py` checks the labels and evidence, and `tests/test_level.py` checks how seniority is read from titles and required years.

> **📁 Files we just created**
> - `hunt/countries.py`: finds countries in messy locations.
> - `hunt/labels.py`: labels each job, saving the exact sentence as evidence.
> - `hunt/registers.py`: UK and Netherlands visa sponsor lists.
> - `hunt/h1b.py`: US visa filing history.
> - `hunt/reach.py`: hides jobs you probably can't get.
> - `tests/test_labels.py` and `tests/test_level.py`: label and seniority tests.

## Step six: how well does it fit?

Now the fit score, in `hunt/scoring.py`, from 0 to 100.

It uses embeddings: lists of numbers that capture what text means. A small, free model called MiniLM makes them, right on your computer. So your CV never leaves your machine. "Built fraud models" and "fraud detection experience" end up close together.

The score mixes five parts: how well the job matches your whole CV (30%), your best project (20%), your skills (20%), the role title (22%) and the field, like fraud or payments (8%). Then it's multiplied by a seniority number. Junior roles keep the full score, and senior roles drop to 45%.

It also writes the "why", the gaps, and your top matching projects. And if MiniLM can't load, it falls back to a simpler word-counting method, so the run still finishes.

Let's pause and look at where we are. We have your profile and settings. We fetch politely, merge copies, keep relevant jobs, label reachability with evidence, and score fit.

That's the brain. Now we store it all, and help you act on it.

> **📁 Files we just created**
> - `hunt/scoring.py`: the relevance filter and the 0 to 100 fit score.

## Step seven: store it, and never lose your notes

`hunt/db.py` stores everything in SQLite, a database in one file. And here's its most important rule. Look at this real line:

```python
USER_FIELDS = ["date_found", "status", "date_applied", "notes", "letter_file", "user_updated_at"]
```

Those are *your* fields. When a daily run updates a job, it never overwrites them. Your status, your notes, your dates stay exactly as you left them.

`hunt/export.py` writes your tracker as an Excel file and a CSV file. And before writing a new one, it reads your edits back in. So you can type notes in Excel, and nothing gets lost.

`hunt/page.py` does the same for an online tracker page, `docs/tracker-page.html`. So you can check and update jobs from your phone, and bring those changes back.

Then `hunt/pipeline.py` ties the full daily run together: fetch, remove copies, filter, label, score, store, export. One broken source never stops the run.

And `hunt/cli.py` gives you the commands: run, list, show, queue, check, mark, note, stats, rescore, verify-companies, page-export, page-import, and cv.

Tests: `tests/test_db.py` checks the "never overwrite my notes" rule. `tests/test_page.py` checks the online page round trip. `tests/test_pipeline.py` runs a full day offline with sample data. And `tests/conftest.py` holds shared test helpers.

> **📁 Files we just created**
> - `hunt/db.py`: the database, which never overwrites your fields.
> - `hunt/export.py`: the Excel and CSV tracker, reading your edits back first.
> - `hunt/page.py`: the online tracker page, both ways.
> - `docs/tracker-page.html`: the online tracker page itself.
> - `hunt/pipeline.py`: the full daily run.
> - `hunt/cli.py`: every command.
> - `tests/test_db.py`, `tests/test_page.py`, `tests/test_pipeline.py`, `tests/conftest.py`: storage and run tests.

## Step eight: help with applications

Now we help you apply, without ever applying for you.

`hunt/queue.py` writes a letter queue: the top 15 reachable jobs, waiting for cover letters.

The letters themselves are drafted outside the code, by an AI chat assistant, following writing rules kept in the project. The project root has an instructions file for that assistant, with the writing rules. The first rule is: only use facts from your CV and projects files, and never invent anything. There's also a saved command that tells the assistant how to work through the newest letter queue.

`docs/chat-instructions.md` is a separate set of instructions for using an AI chat to find jobs, check a job, or write an application pack. It says clearly: "I review everything and apply myself."

Then the safety net: `hunt/checker.py`. It checks every letter. Any number that isn't in your CV or projects gets flagged, so made-up facts get caught.

It also flags banned phrases, dash characters, wrong length (150 to 250 words) and a missing sign-off. Tested in `tests/test_checker.py`.

And `hunt/cv.py` builds a tailored CV for each job, as a Word file and a PDF, from `profile/cv_data.yaml`. Same facts, but the headline, summary, project order and skill order are picked for the job. One column, standard headings, so job systems can read it.

## What the tool has produced so far

The project folder also holds what it's made. `output/letters/` has about 67 drafted cover letters, one per job, each named by date, company and role, like `2026-09-27_adyen_merchant-fraud-analyst.md`. A small `.gitkeep` file keeps the folder there even when it's empty.

`output/cvs/` has the tailored CVs, a Word and a PDF for each job. And `output/cvs/specs/` has a small file per job, recording which headline, summary and project order were chosen for it.

There's one more document, `docs/internships.md`. It's a list of paid, remote internships and programs open to you, like the Cohere Labs Scholars Program and Google Summer of Code, with when to apply. They're programs, not single jobs, so they're not in the tracker.

> **📁 Files we just created**
> - `hunt/queue.py`: writes the letter queue.
> - The assistant's instructions file and its saved letters command: the writing rules and how to draft from the queue.
> - `docs/chat-instructions.md`: instructions for using an AI chat for the job hunt.
> - `hunt/checker.py`: checks every letter for made-up numbers and banned phrases.
> - `tests/test_checker.py`: tests the letter checker.
> - `hunt/cv.py`: builds a tailored CV per job.
> - `output/letters/`: about 67 drafted letters, plus `.gitkeep`.
> - `output/cvs/`: a Word and PDF CV for each job.
> - `output/cvs/specs/`: what was tailored for each job's CV.
> - `docs/internships.md`: paid remote programs open to you.

## So, how's it doing?

74 tests pass in about 1 second, all offline. They cover removing copies, labels, seniority, the "never overwrite my notes" rule, spreadsheet and page edits, the letter checker, and a full offline run.

But whether high fit scores actually lead to interviews **hasn't been measured yet**. There's no data on outcomes so far. How accurate the labels are hasn't been measured either.

## What's still missing?

- **The fit score's weights were set by hand**, not learned from results.
- **Text rules can miss unusual wording**, so some jobs will be mislabelled.
- **Being on a sponsor list doesn't guarantee** a particular job sponsors.
- **Meaning-matching catches the topic**, not how senior or deep a job is.
- **It fetches one source at a time**, so it slows down as sources are added.
- **It holds personal data** like CVs and letters, so the project must stay private.

## Let's put it all together

So let's look at it in one breath.

We **set up the workshop**, then wrote down **your profile** first, because it's the measuring stick for everything. We put every setting in **YAML files**, including the 405 companies. We **fetched politely**, turned every job into **one shape**, and **health-checked** the boards.

We **merged copies** and kept only **relevant jobs**. We **labelled reachability** with saved evidence, using visa sponsor lists for the UK, Netherlands and US. We **hid jobs out of reach**, but kept them. We **scored fit** with a free, local model.

Then we **stored** everything without ever overwriting your notes, built a **two-way tracker** in Excel and online, and added **help with applications**: a queue, a letter checker that stops made-up numbers, and tailored CVs.

Notice how it links. Your profile from step one feeds the scorer, the letter checker and the tailored CVs. The evidence from the labeller lets you check every decision. And the "never overwrite" rule means you can run it every single day, safely.

That's the project. It finds the jobs, and you stay in charge.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
