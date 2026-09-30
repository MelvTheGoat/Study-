# Job Hunt Helper: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow the path of a job through the system: collecting it, cleaning it, labelling it, scoring it, and tracking it.

---

## 1. Collecting Jobs

### ATS (applicant tracking system)
**What it means:** software companies use to post jobs and collect applications. Greenhouse, Lever and Ashby are examples.

**Why it's needed here:** a company's own ATS board has the original, fullest job post. The project reads 405 company boards across 9 ATS systems.

### Job board and aggregator
**What it means:** a job board lists jobs from many companies. An aggregator collects jobs from other sites.

**Why it's needed here:** they add jobs from companies not on the list. LinkedIn and Indeed are left out because they don't allow automated collection.

### API
**What it means:** a way for one program to ask another for data, by sending a request to a web address.

**Why it's needed here:** most ATS systems and job boards share their jobs through an API, which is cleaner than reading web pages. Some need a free key, and those only run if you've added one.

### API key
**What it means:** a secret code that identifies you to a service.

**Why it's needed here:** a few optional sources (like Adzuna and Reed) need one. Keys are kept in a separate private file, so they never end up in the shared code.

### Rate limiting (polite fetching)
**What it means:** deliberately slowing down how often you send requests to the same website.

**Why it's needed here:** too many fast requests can get you blocked, and some sources ask for it. For example, Remotive asks for no more than four calls a day, so its results are saved and reused for some hours.

### Retries with backoff
**What it means:** trying a failed request again, waiting a little longer each time.

**Why it's needed here:** many failures are temporary. Waiting longer each time gives the website a chance to recover.

### User-Agent
**What it means:** a short label a program sends with each request, saying who it is.

**Why it's needed here:** a clear label is polite and honest. Websites can see who's visiting and why.

### Failure isolation
**What it means:** making sure one broken part can't stop everything else.

**Why it's needed here:** with hundreds of sources, some will fail on any day. Each source runs on its own, and errors are collected and shown at the end.

### Board health check
**What it means:** testing every company board to see if it still works.

**Why it's needed here:** companies change ATS systems or close boards. The check tries other ATS systems for dead boards and moves broken ones to a "removed" list.

---

## 2. Cleaning the Jobs

### Deduplication (dedupe)
**What it means:** finding and merging copies of the same thing.

**Why it's needed here:** one job can appear on several boards. Posts with the same apply link, or the same company, title and location, are merged, keeping the longest description.

### Hash
**What it means:** a short fingerprint made from some data. The same data always gives the same fingerprint.

**Why it's needed here:** the company, title and location are turned into a fingerprint, so copies can be matched quickly.

### Relevance filter
**What it means:** a quick check that throws away items that clearly don't fit.

**Why it's needed here:** most jobs aren't about data, ML or AI. Dropping them early saves time before the slower scoring step. Vague titles like "engineer" must mention enough ML words in the description to stay.

### Regular expression (regex)
**What it means:** a pattern for finding text, like "any title containing 'machine learning' or 'data scien'".

**Why it's needed here:** the relevance filter, seniority check and reach rules all use regex patterns. They're simple and fast, but can miss unusual wording.

---

## 3. Labelling Reachability

### Location label
**What it means:** a tag saying whether a job is open to you: remote worldwide, in Nigeria, in Africa, with visa sponsorship, or restricted.

**Why it's needed here:** a perfect job you can't legally take is useless. The label decides whether a job shows up in your list.

### Visa sponsorship
**What it means:** when an employer helps you get permission to work in their country.

**Why it's needed here:** for jobs abroad, sponsorship is what makes them reachable from Nigeria.

### Sponsor register
**What it means:** an official government list of companies allowed to sponsor work visas.

**Why it's needed here:** a company on the UK or Netherlands list probably sponsors, even if the post doesn't say so. The lists are refreshed weekly.

### H-1B filings
**What it means:** records of US work visas that companies have applied for.

**Why it's needed here:** a US company with at least 5 filings, including one in the last 3 years, is treated as a likely sponsor. The results are saved for 30 days.

### ECOWAS
**What it means:** the Economic Community of West African States, a group of countries including Nigeria.

**Why it's needed here:** Nigerians don't need a visa to work in these countries, so jobs there count as reachable.

### Evidence
**What it means:** the exact sentence a rule used to make its decision.

**Why it's needed here:** rules can be wrong. Saving the sentence (like "must be based in the US") lets you check each label yourself.

### Reach rules
**What it means:** rules that hide jobs you probably can't get: restricted, too senior, 5+ years, students-only, or needing a Master's or PhD.

**Why it's needed here:** they keep your list realistic. Hidden jobs stay in the database, so they don't come back as "new" next time.

---

## 4. Scoring the Fit

### Embedding
**What it means:** a list of numbers that captures what a piece of text means. Similar meanings give similar numbers.

**Why it's needed here:** it lets the tool compare your CV with a job by meaning, not just matching words. "Built fraud models" and "fraud detection experience" end up close together.

### Embedding model (MiniLM)
**What it means:** a program that turns text into embeddings. MiniLM is a small, free one that runs on a normal computer.

**Why it's needed here:** it's free, private and fast enough. Your CV never leaves your computer.

### Cosine similarity
**What it means:** a number showing how close two embeddings point in the same direction. Closer means more similar meaning.

**Why it's needed here:** it's the "how similar is this job to my CV?" number. Raw values bunch up in a narrow band, so values between 0.30 and 0.70 are stretched to cover 0 to 1.

### Chunking
**What it means:** splitting long text into smaller pieces.

**Why it's needed here:** the model only reads a limited amount of text at once. Jobs are split into up to 4 chunks of 180 words, so a long post isn't cut off after the first part.

### Cache
**What it means:** a saved copy of a result, reused instead of working it out again.

**Why it's needed here:** embeddings are saved in the database. Tomorrow's run doesn't need to re-make embeddings for text it has already seen, which makes it much faster.

### Fallback
**What it means:** a simpler backup used when the main method fails.

**Why it's needed here:** if MiniLM can't load, a basic word-counting method is used instead. The ranking is rougher, but the run still finishes.

### Weighted average
**What it means:** an average where some parts count more than others.

**Why it's needed here:** the fit score mixes five parts: CV match (30%), best project (20%), skills (20%), role (22%) and field (8%).

### Level multiplier
**What it means:** a number the score is multiplied by, based on how senior the job is.

**Why it's needed here:** senior jobs shouldn't top your list. Junior and entry roles keep their full score (×1.0), while senior roles sink (×0.45).

### Heuristic
**What it means:** a sensible rule of thumb, set by a person, rather than learned from data.

**Why it's needed here:** the fit score is a heuristic. Its weights are hand-set in a settings file, and whether high scores lead to interviews isn't measured yet.

---

## 5. Tracking and Writing

### Upsert
**What it means:** "update or insert": update a row if it exists, or add it if it doesn't.

**Why it's needed here:** re-running daily updates job details. Your own fields (status, notes, date applied, letter file) are never overwritten.

### Two-way sync
**What it means:** changes made in either place are copied to the other.

**Why it's needed here:** you can edit the spreadsheet or the online tracker page, and your edits are read back into the database before a new copy is written.

### CLI (command-line interface)
**What it means:** a program you control by typing commands, like `python hunt.py run`.

**Why it's needed here:** each daily step is one short command: run, queue, check, mark, note and so on.

### Letter queue
**What it means:** a list of the top jobs waiting for a cover letter.

**Why it's needed here:** it's the hand-off from finding jobs to writing letters. By default it picks the top 15 reachable jobs.

### Letter checker
**What it means:** an automatic proofreader for cover letters.

**Why it's needed here:** it flags any number that isn't in your CV, banned phrases, dash characters, a wrong length (150 to 250 words by default) and a missing sign-off. This stops made-up facts.

### Human in the loop
**What it means:** a person checks and approves each important step.

**Why it's needed here:** the tool never applies for you. You read every letter and apply yourself, which keeps quality high and respects each site's rules.

### ATS-friendly CV
**What it means:** a CV laid out so application software can read it: one column and standard headings.

**Why it's needed here:** the tool builds a tailored CV for each job from one file of facts. It reorders what you've done to suit the job, but never invents anything.
