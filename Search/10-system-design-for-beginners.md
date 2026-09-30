# Job Hunt Helper: System Design for Beginners

This project is a personal job-search helper for ML and AI roles, built for someone applying from Nigeria. It collects jobs from hundreds of company websites, removes copies, works out whether each job is actually open to you, and scores how well it matches your CV. It never applies for you. It just makes sure you read the best, reachable jobs first.

## Key Terms

- **Job board**: a website that lists jobs, like RemoteOK.
- **ATS (applicant tracking system)**: the software companies use to post jobs and collect applications, like Greenhouse or Lever. Each company has its own board on one.
- **Duplicate**: the same job appearing in more than one place.
- **Visa sponsorship**: when an employer helps you get permission to work in their country.
- **Embedding**: a list of numbers that captures what a piece of text means. Texts with similar meanings get similar numbers.
- **Fit score**: a number from 0 to 100 for how well a job matches your CV.
- **Database**: an organised store of data in tables, like a set of linked spreadsheets.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The goal isn't "find every job". It's "find jobs I can actually get, and read the best ones first".

**Step 2: Figure out the data.** Use free, official sources: company career boards and job sites that allow it. Sites that forbid automated collection, like LinkedIn, are left out.

**Step 3: Sketch the main parts.** Jobs are fetched, cleaned of copies, labelled by reachability, scored, and saved. A spreadsheet shows the results.

**Step 4: Walk through one daily run.** Follow a job from a company's board to a row in your tracker.

**Step 5: Decide how to know it works.** Check that re-runs never lose your notes, and that the top of the list really is relevant.

**Step 6: Plan for problems.** Websites change, sources go down, and rules can mislabel jobs. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Check **405** company boards across **9** ATS systems, plus several job boards.
- Run **once a day** on your own computer.
- Score every relevant job from **0 to 100**.
- **Never** overwrite your own notes or statuses.

How is a fit score made? Step by step, with example numbers:

1. Five parts are each scored from 0 to 1: CV match 0.6, best project 0.5, skills 0.5, role 1.0, field 0.5.
2. Each is multiplied by its weight (0.30, 0.20, 0.20, 0.22 and 0.08), which add up to 1.
3. 0.18 + 0.10 + 0.10 + 0.22 + 0.04 = **0.64**.
4. This is multiplied by a level score (1.0 for the right seniority), then by 100: fit = **64**.

### The Big Picture (Step 3)

```
   [Job sources: 405 companies + job boards]
                  |
                  v
   [Polite Fetcher]
                  |
                  v
   [Remove copies + keep ML/AI jobs]
                  |
                  v
   [Labeller: can I get this job?]
                  |
                  v
   [Scorer: how well does it fit my CV?] ---> [Database]
                                                  |
                                                  v
                                [Tracker spreadsheet + Letter queue]
```

Step by step:

1. You run one command (or your computer runs it on a timer).
2. The Polite Fetcher collects jobs from every source. If one source breaks, the others carry on.
3. Copies are merged, keeping the longest description, and jobs that aren't about data, ML or AI are dropped.
4. The Labeller decides whether each job is open to you: remote, in Nigeria or Africa, or with signs of visa sponsorship.
5. The Scorer gives each job a fit score from 0 to 100, with the reasons and any gaps.
6. Everything is saved to the database, without touching your own notes.
7. A fresh tracker spreadsheet is written, and the top 15 reachable jobs go to a letter queue.

### The Main Parts (Step 3)

**Polite Fetcher.** It waits between requests to the same website, retries with growing waits, and says clearly who it is. It's like a polite visitor who knocks, waits, and doesn't bang on the door. Being polite stops you getting blocked.

**Remove Copies.** The same job often appears on several boards. Two posts with the same apply link, or the same company, title and location, count as one. It's like merging two contacts on your phone for the same person.

**Labeller.** It reads the job text for clues, like "must be based in the US" or "remote worldwide". It also checks public lists of companies that sponsor visas in the UK, the Netherlands and the US. It saves the exact sentence it used, so you can check its reasoning, like a teacher showing their working.

**Scorer.** It compares your CV and projects with each job using embeddings, made by a small, free model called MiniLM that runs on your own computer. It also checks skills, role and field. It's like a friend who knows your CV well, reading each job and saying "this one's a 64".

**Reach Rules.** Jobs that are too senior, need 5+ years, need a Master's or PhD, or are closed to you are hidden but kept. That way they don't come back as "new" tomorrow, like putting junk mail in a drawer instead of the post tray.

**Database and Tracker.** Your status and notes are never overwritten by a re-run. You can edit the spreadsheet, and your edits are read back in before a new one is written.

**Letter Checker.** Cover letters are drafted outside the code. The checker then flags any number that isn't in your CV, banned phrases, dash characters and wrong length. It's like a proofreader who checks you haven't made anything up.

A small trained classifier could replace the text rules one day (advanced - skip for now).

### How We Know It's Working (Step 5)

- **Tests**: 74 automatic checks pass in about 1 second. They cover removing copies, labels, the "never overwrite my notes" rule, spreadsheet edits and the letter checker.
- **Reachable jobs per day**: how many jobs make it into the tracker after the rules.
- **Source errors**: which sources failed on each run, so they can be fixed.
- **Ranking quality**: whether high scores lead to interviews. **Not measured yet**, because there's no data on outcomes.

### What Can Go Wrong (Step 6)

- **Company boards move or die.** A health check tests every board, tries other ATS systems for dead ones, and moves broken ones to a "removed" list.
- **A source goes down.** Each source runs separately, so one failure can't stop the whole run.
- **Text rules mislabel a job.** Unusual wording can fool them. That's why the exact sentence is saved as evidence.
- **Personal data.** The project holds CVs and letters, so it must stay private.

## Quick Recap

- Collect jobs politely from free, official sources only.
- Merge copies, then keep only relevant jobs.
- Label reachability with evidence you can check.
- Score fit locally with embeddings plus simple rules.
- Never overwrite your notes, and never apply automatically.
