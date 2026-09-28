# Search (Job Hunt): How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Search (private)

---

## Step 1: Requirements (2 min)

One line:
> "A daily pipeline that finds ML/AI jobs I can actually get from Nigeria, ranks them against my CV, and tracks my applications."

**Functional**
1. Fetch jobs from company boards (ATS) and job boards.
2. Remove duplicates.
3. Keep only data/ML/AI jobs.
4. Label reachability: remote-open, Nigeria, Africa, visa sponsor, unknown, restricted.
5. Score fit 0–100 against my CV and projects, with a reason and skill gaps.
6. Track status and notes, in a spreadsheet and an online page.
7. Queue top jobs for letters, and check the letters.

**Non-functional**
- **Free:** no paid APIs, runs on a laptop CPU.
- **Polite:** rate limits and caching per source. Only sources that allow it.
- **Safe re-runs:** never overwrite my status, notes or dates.
- **Never applies automatically.**

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Company boards | 405 in `companies.yaml` |
| ATS types | 9 |
| Embedding model | all-MiniLM-L6-v2, ~90 MB, 384-dim, CPU |
| Chunks per job | up to 4 × 180 words |
| Score weights | CV 0.30, project 0.20, skills 0.20, role 0.22, domain 0.08 |
| Out-of-reach years | 5+ |
| Letter length | 150–250 words (checker default) |
| Tests | 74 |

**Say:** "Hundreds of boards, thousands of jobs a day at most. One laptop is enough."

---

## Step 3: High-level boxes (2 min)

```
[Company boards] [Job boards] [Keyed APIs]
          \           |           /
           [Polite HTTP client + cache]
                      |
   [Dedupe] -> [Relevance filter] -> [Labeller <- sponsor registers, H-1B] -> [Scorer (MiniLM + rules)]
                                                                                  |
                                                                            [SQLite jobs.db]
                                                                                  |
                          [Reach rules] -> [tracker.xlsx / page]   [Letter queue -> checker]
```

---

## Step 4: Deep dive (8–10 min)

### 4a. Fetching
- One module per ATS API, plus board scrapers/APIs.
- **Isolation:** a broken source adds an error and the run continues.
- Per-host minimum interval, retries with backoff, caching for aggregators.
- `verify-companies` repairs the board list.

### 4b. Dedupe
- Keys: apply URL, and a hash of company + title + location.
- Keep the longer description (ATS copy beats aggregator snippet).

### 4c. Labelling
- Find countries, regions and cities in the location line and text.
- Scan sentences for "no sponsorship", "right to work", "citizens only", "clearance", "location only" and positive sponsorship phrases.
- Add evidence from the UK/NL registers and US H-1B filings.
- Output: label + the exact restricting sentence.

### 4d. Scoring
Write the formula:
```
base = (0.30·cv_sim + 0.20·project_sim + 0.20·skills + 0.22·role + 0.08·domain) / 1.0
fit  = 100 × base × level_multiplier
```
- Similarities: cosine of mean chunk embeddings, mapped from [0.30, 0.70] to [0, 1].
- Skills: `(matched + 0.5·2) / (job_skills + 2)`, so small skill lists get pulled toward 0.5.
- Level: intern/grad/junior/entry 1.0, mid 0.8, senior 0.45, lead 0.4, staff/principal/manager 0.35.

### 4e. Storage and sync
- Upsert by job key. User fields (status, notes, dates, letter file) are never overwritten.
- Read Excel edits back before re-writing. Import page edits.

### 4f. Letters
- Queue → an AI assistant drafts → `check` flags numbers not in the CV, dashes, banned phrases, length and sign-off → fix → `mark-drafted`.

---

## Step 5: Bottlenecks (2 min)

1. **Source breakage:** API changes or dead boards. Handled by isolation, `verify-companies` and error reporting.
2. **Embedding time:** batched and cached in SQLite.
3. **Rate limits:** per-host waits and cached aggregators.
4. **Label errors:** regex misses. Handled by keeping evidence sentences and a separate restricted tab.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Local MiniLM | Paid LLM scoring | Free, fast, private | Shallow matching |
| Hand-set weights | Learned ranking | No outcome data yet | Unvalidated ranking |
| Regex labels | ML classifier | Transparent, evidence kept | Misses unusual wording |
| SQLite + Excel | Web app + DB | Zero setup | Single user |
| Human applies | Auto-apply | Quality, and respects sites' rules | Slower |
| Official sources only | LinkedIn/Indeed scraping | Their terms forbid it | Less coverage |
