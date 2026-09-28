# I built a job-hunting pipeline that asks "can I actually get this job?"

Repo: https://github.com/MelvTheGoat/Search (private)

## Why I built it

Looking for ML and AI jobs from Nigeria is slow in a way that isn't obvious from outside. It isn't just that there are lots of jobs to read. Many of the jobs that look perfect are out of reach: "must have the right to work in the UK", "US citizens only", "remote, but within Germany". You find that out in the last paragraph, after reading the whole post.

I wanted a tool that does the first pass for me every morning:

1. Fetch new jobs from sources that allow it.
2. Remove duplicates.
3. Keep only data, ML and AI roles.
4. Tell me whether I could realistically get each one.
5. Rank them against my CV.
6. Keep a tracker that never loses my notes.

And it had to be free, run on a laptop, and **never apply for me**.

## The problem

The hard parts aren't the ones you'd expect. Fetching jobs is mostly plumbing. The hard parts are:

- **Duplicates:** the same job shows up on the company board and on three aggregators.
- **Work rights:** they hide in free text, phrased a hundred ways.
- **Ranking:** "this job fits my CV" has to be fast, free and private.
- **Not losing my own data** when the tool re-runs every day.

## How it works

### Sources: official and polite

The tool reads company job boards on nine applicant tracking systems (Greenhouse, Lever, Ashby, SmartRecruiters, Workable and others), from a list of 405 boards that includes more than 60 African companies. It also reads remote job boards, a few aggregators, Amazon's own search for African countries, and the monthly Hacker News "Who is hiring?" thread. A few more sources switch on only if you add a free API key.

LinkedIn and Indeed aren't used. They don't allow scraping.

Every request has a clear User-Agent, waits between calls to the same site, and retries with growing waits on errors. Aggregator results are cached for a few hours, because Remotive, for example, asks for no more than four calls a day.

One broken source never stops the run:

```python
try:
    found = fetch_company(http, c, keep_title=keep_title)
except Exception as e:  # noqa: BLE001
    errors.append(f"{c['ats']}/{c['token']}: {e}")
    continue
```

Company boards move and die, so `verify-companies` tests every board, tries the same name on other ATSs when a token is dead, and moves gone boards to a separate file.

### Dedupe: keep the better copy

Two jobs are the same if they share an apply URL, or the same company + title + location. When there are two copies, the one with the longer description wins, so the full company post beats a short aggregator snippet.

### Labels: can I get this job?

This is the heart of it. Each job gets one label, best first:

| Label | Meaning |
|---|---|
| `remote_open` | Remote, and open to Nigeria, Africa, EMEA or worldwide |
| `nigeria` | Onsite or hybrid in Nigeria |
| `africa` | Onsite or hybrid elsewhere in Africa (ECOWAS countries need no visa for Nigerians) |
| `sponsor_yes` | Abroad, and the post offers a visa or relocation |
| `sponsor_likely` | Abroad, and the company is a known sponsor |
| `sponsor_unknown` | Abroad, and the post says nothing |
| `restricted` | No sponsorship, existing right to work, one country only, citizenship or clearance |

The labeller splits each post into sentences and scans them against phrase lists in `config/sponsorship.yaml`. When it finds a restriction, it saves **the exact sentence**, so I can check it in seconds rather than trust the tool blindly.

For "known sponsor", it uses three public sources: the UK Register of Licensed Sponsors, the Netherlands IND register (both refreshed weekly), and US H-1B filing history. A company counts as a US sponsor with at least five filings under its own name and at least one in the last three years.

Some jobs are then out of reach: restricted, abroad with no sign of sponsorship, senior roles, posts asking for 5+ years, students only, or a Master's or PhD as a must. They stay in the database, so they don't come back as "new" tomorrow, but they're left out of my list.

### Scoring: fast, free and local

Each job gets a fit score from 0 to 100, with no AI API calls. It's a weighted average of five parts, times a level multiplier:

| Part | Weight | How |
|---|---|---|
| CV similarity | 0.30 | Cosine similarity of MiniLM embeddings: job vs my whole CV |
| Project similarity | 0.20 | Job vs my best-matching project |
| Skills | 0.20 | Share of the job's skills that are in my CV |
| Role | 0.22 | Target titles 1.0, related 0.5, other 0.25 |
| Domain | 0.08 | Fraud, risk, credit, payments, fintech |

Then it's multiplied by the level: intern to junior keep 100%, mid keeps 80% (and is flagged "stretch"), and senior to manager keep 35–45%, so they sink but stay visible.

One small detail I like: when a post names only a few skills, one lucky match would give 100%. So the skills score is pulled toward 0.5:

```python
prior = self.cfg.get("skills_prior_weight", 2)
skill_score = (len(matched) + 0.5 * prior) / (len(job_skills) + prior)
```

Embeddings come from `all-MiniLM-L6-v2` on CPU, about 90 MB. Each job is split into up to four 180-word chunks, averaged, and cached in SQLite, so re-runs are fast. If the model can't load, it falls back to a simple word-hash vector and says so.

The scorer also writes a one-line "why", the skill gaps, and the top matching projects, so the number is never alone.

### The tracker: never lose my notes

Everything goes into a SQLite database and out to `tracker.xlsx` and `tracker.csv`. The rule: **a re-run never changes my status, notes, date applied, letter file, or the date a job was first found.** I can edit status and notes right in Excel. The next run reads my edits back before it rewrites the file. There's also an online tracker page with a two-way sync.

### Letters: checked, not trusted

`hunt.py queue --top 15` writes a queue of the best jobs in reach. The letters are then drafted by an AI assistant in a chat session, following writing rules in the repo. That part isn't Python.

What *is* Python is the checker, and I think it's the most important safety feature. `hunt.py check` flags:

- any **number** in a letter that isn't in my CV or projects file
- dash characters and banned phrases
- a letter outside 150–250 words
- a missing sign-off

The number check catches the most dangerous kind of mistake: a letter that claims a result I never achieved.

Each job with a letter also gets a tailored CV (Word and PDF), built from one YAML file of facts. The facts never change. Only the headline, summary, project order and skill order are picked for the job.

## What I learned

- **Reachability matters more than fit.** A 95% match you can't legally take is worth zero.
- **Keep evidence, not just labels.** Saving the exact sentence makes regex rules trustworthy enough to use.
- **Re-runs must be safe.** The "never overwrite my fields" rule is what makes a daily tool usable.
- **Check AI output with plain code.** A simple "is every number in my CV?" check stops invented claims.
- **Isolate failures.** With hundreds of sources, something always breaks. The run must carry on.

## What's next

- **Measure the score.** Right now the weights are hand-set. Once I have enough applications, I can check whether a higher fit score actually leads to more interviews. That isn't measured yet.
- **Better labels.** Replace some regex rules with a small classifier trained on labelled sentences.
- **Parallel fetching**, to cut the daily run time.
