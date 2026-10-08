# Job Hunt Helper: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

## 1. The Problem

Finding machine learning and AI jobs from Nigeria is hard work. You have to check hundreds of company career pages and job websites every day.

Then you have to work out which jobs you can actually get: remote and open worldwide, in Nigeria or Africa, or with help getting a visa. Most job posts don't say clearly.

## 2. The Big Idea

This project is a personal job-search helper that runs on your own computer. Every day it collects jobs, removes copies, works out which ones are open to you, and scores how well each one fits your CV.

It never applies for you. A person reads every job and every cover letter, and applies themselves. The helper just makes sure you read the best, reachable jobs first.

## 3. How It Works, Step by Step

**Step 1: Collect jobs from free, official sources.** It checks 405 company job boards. These run on 9 different systems that companies use to post jobs, like Greenhouse and Lever. It also checks several job websites that allow automatic collection.

Since October it also looks at startups. It reads the Y Combinator job board, and once a week it builds a list of startups from public startup lists (a16z, the Breakout List, Next Play and Ramp) and finds each one's job board. Startup jobs get their own tag and tab.

It deliberately skips LinkedIn and Indeed, because they don't allow it.

**Step 2: Be polite while collecting.** It waits between requests to the same website, says clearly who it is, and retries failures with growing waits. If one source breaks, the others carry on. And if a website keeps saying "too many requests", it's skipped for the rest of that run, so one busy site can't hold everything up.

**Step 3: Remove copies.** The same job often appears on several websites. Two posts with the same apply link, or the same company, title and location, count as one. The one with the longest description is kept.

**Step 4: Keep only relevant jobs.** Jobs that clearly aren't about data, machine learning or AI are dropped. This saves time before the slower steps.

**Step 5: Work out if you can get the job.** It reads the job text for clues, like "must be based in the US" or "remote worldwide". It also checks official lists of companies that sponsor visas in the UK, the Netherlands and the US. It saves the exact sentence it used, so you can check its reasoning.

**Step 6: Score how well it fits.** It gives each job a fit score from 0 to 100, with reasons and any gaps.

**Step 7: Hide jobs out of reach.** Jobs that are too senior, need 5+ years, need a Master's or PhD, or are closed to you are hidden. They're kept in the database, so they don't come back as "new" tomorrow.

**Step 8: Update your tracker.** Everything goes into a spreadsheet and an online tracker page. Your own notes and statuses are never overwritten. The page now also has a Startups tab, and an Outreach section for keeping track of people you contact, with message templates and follow-ups.

**Step 9: Prepare for cover letters.** The top 15 reachable jobs go into a letter queue. The letters are drafted outside the code, then the code checks them.

## 4. The Clever Parts

**Scoring by meaning, for free.** It compares your CV with each job using embeddings. An embedding is a list of numbers that captures what text means. "Built fraud models" and "fraud detection experience" end up close together.

It uses a small, free model called MiniLM that runs on your own computer. So your CV never leaves your machine.

**A clear recipe for the fit score.** The score mixes five parts: how well the job matches your CV (30%), your best project (20%), your skills (20%), the role title (22%) and the field (8%). Then it's multiplied by a seniority number. Junior roles keep their full score, while senior roles drop to 45%.

**Evidence you can check.** Every reachability label saves the sentence it was based on. Rules can be wrong, so you can always see why.

**Never losing your notes.** You can edit the spreadsheet in Excel. Before writing a new one, the helper reads your edits back in. Re-running every day is always safe.

**A letter checker that stops made-up facts.** It flags any number in a cover letter that isn't in your CV. It also flags banned phrases, dash characters, a wrong length (150 to 250 words) and a missing sign-off.

**Tailored CVs without inventing.** For each job, it builds a CV in Word and PDF from one file of facts about you. It reorders your projects and skills to suit the job, but never invents anything.

## 5. The Important Words

- **ATS (applicant tracking system)**: software companies use to post jobs, like Greenhouse.
- **Job board**: a website that lists jobs.
- **Duplicate**: the same job appearing in more than one place.
- **Visa sponsorship**: when an employer helps you get permission to work in their country.
- **Embedding**: a list of numbers that captures what text means.
- **Fit score**: a number from 0 to 100 for how well a job matches your CV.
- **Heuristic**: a sensible rule of thumb set by a person, not learned from data.
- **Upsert**: update a row if it exists, or add it if it doesn't.
- **Regex**: a text pattern used to find words or phrases.
- **Human in the loop**: a person checks and approves each important step.

## 6. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **requests**: fetches jobs from websites.
- **sentence-transformers and MiniLM**: compare your CV with jobs by meaning.
- **NumPy**: does the maths for comparing.
- **SQLite**: a simple database in one file that stores everything.
- **openpyxl**: writes and reads the Excel tracker.
- **python-docx**: builds the tailored CVs.
- **PyYAML**: holds all the settings, so you can tune it without touching code.
- **cron or Task Scheduler**: runs the search daily.
- **pytest**: runs 84 automatic checks.

## 7. How Good Is It?

84 automatic tests pass in a few seconds. They cover removing copies, labels, the "never overwrite my notes" rule, spreadsheet edits and the letter checker.

But whether high fit scores actually lead to interviews **hasn't been measured yet**. The tool has run every day since late September, with about 126 cover letters drafted by 8 October, but interview results aren't recorded yet. How accurate the reachability labels are hasn't been measured either.

## 8. What's Weak or Missing

- The fit score's weights were set by hand, not learned from results.
- The text rules can miss unusual wording, and some jobs will be mislabelled.
- Being on a visa sponsor list doesn't guarantee a particular job sponsors.
- Meaning-matching catches the topic of a job, but not how senior or deep it is.
- It collects from one source at a time, so it gets slower as sources are added.
- It holds personal data like CVs and letters, so the project must stay private.

## 9. What This Project Shows You Can Do

- Build a useful tool that solves your own real problem.
- Collect data politely and only from allowed sources.
- Use AI models locally and for free, keeping data private.
- Design rules that show their evidence, so they can be checked.
- Keep a human in charge of important decisions.

## 10. Ten Things to Remember

1. It's a personal job-search helper for ML and AI roles, applying from Nigeria.
2. It checks 405 company boards plus several job websites, every day.
3. It only uses free, official sources that allow it.
4. It removes copies and keeps only relevant jobs.
5. It labels whether each job is open to you, with evidence.
6. It scores fit from 0 to 100, using meaning plus simple rules.
7. Jobs out of reach are hidden but kept.
8. Your notes are never overwritten.
9. A checker stops made-up numbers in cover letters.
10. It never applies for you.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
