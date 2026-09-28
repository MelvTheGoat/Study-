# Search (Job Hunt): Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Search (private)

---

## Summary

A single-user Python CLI (~3,800 lines in `hunt/`) that fetches jobs from 405 ATS company boards and a set of job boards, dedupes, filters for data/ML/AI, labels work-rights reachability with evidence, scores fit locally with MiniLM embeddings plus rules, stores everything in SQLite, and syncs a spreadsheet and an online tracker. It also builds tailored CVs and checks externally drafted cover letters. **74 tests pass (~1 s).** 71 commits over 26–27 Sep 2026.

## Architecture

```
hunt.py                 entry point -> hunt/cli.py
hunt/http.py            Http: UA, per-host min interval, retries, backoff, timeout
hunt/sources/ats.py     Greenhouse, Lever, Ashby, SmartRecruiters, Workable,
                        Recruitee, Personio, BambooHR, Breezy
hunt/sources/boards.py  RemoteOK, Remotive, Arbeitnow, Himalayas, Jobicy,
                        Working Nomads, WWR (RSS), The Muse, HN, Amazon,
                        + keyed: Adzuna, Reed, Jooble, Findwork
hunt/sources/__init__   time-based cache for aggregators
hunt/verify.py          board health check + auto-repair
hunt/dedupe.py          URL / company+title+location keys
hunt/countries.py       country, region, city detection
hunt/labels.py          Labeller: location label, restriction sentence, sponsorship
hunt/registers.py       UK + NL sponsor registers (weekly)
hunt/h1b.py             US H-1B filings (30-day cache)
hunt/scoring.py         is_relevant, detect_level, Skills, Embedder, Scorer
hunt/reach.py           out_of_reach rules
hunt/db.py              SQLite schema, upsert that keeps user fields
hunt/export.py          xlsx/csv export + read-back of edits
hunt/page.py            online page export/import
hunt/queue.py           letter queue
hunt/checker.py         letter checks
hunt/cv.py              tailored CV (docx + pdf) from YAML
config/*.yaml           companies, scoring, skills, sources, sponsorship, writing
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| Local embeddings | `all-MiniLM-L6-v2`, CPU, 384-dim, normalised | Free, private, fast enough |
| Embedding cache | SQLite, key = hash(model tag + text) | Re-runs skip re-embedding. The model change is in the key. |
| Fallback vectoriser | 4,096-dim hashed unigrams + bigrams if the model fails | The run still produces a rougher ranking. It logs a warning and changes the similarity scale. |
| Chunked docs | ≤4 chunks × 180 words per job (CV up to 12), mean then normalise | Long posts don't get truncated to the first part only |
| Similarity scaling | Linear map of cosine from [0.30, 0.70] to [0, 1] | Raw MiniLM cosines cluster in a narrow band |
| Skills prior | `(matched + 1) / (n + 2)` | A 1-skill post can't score 100% |
| Level multiplier | Title regex first, then "looking for a senior", then required years | Titles are the clearest signal |
| Evidence-first labels | Store the restricting sentence and sponsor evidence | Human-checkable, debuggable rules |
| Soft hiding | Out-of-reach jobs stay in the DB but are hidden | They don't reappear as "new" |
| User-owned fields | Upsert never touches status, notes, date applied, letter file, date found | Safe daily re-runs |
| Failure isolation | Per-source try/except, errors collected | One dead board can't kill the run |
| Human in the loop | No auto-apply. Letters drafted outside the code, then checked. | Quality, and respects site terms |

## Algorithms in detail

**Relevance:** drop by `drop_titles` → keep by `keep_titles` → for generic engineer titles, require ≥ `min_ml_words` ML words in the description.

**Level detection:** ordered regexes (intern, manager, principal, staff, lead, senior, graduate, entry, junior, mid). Then phrases like "looking for a senior". Then `required_years()` (prefers the bachelor's figure when degrees give different years): ≥5 → senior, ≥3 → mid (stretch), otherwise junior.

**Fit score:**
```
parts = {cv_similarity, project_similarity, skills, role, domain}   # each in [0,1]
base  = Σ w_k · part_k / Σ w_k       # w = 0.30, 0.20, 0.20, 0.22, 0.08
fit   = round(100 · base · level_score, 1)
```
Domain = min(1, hits/2), where a hit is a title match or ≥2 mentions in the description.

**Reach:** out of reach if label ∉ {remote_open, nigeria, africa, sponsor_yes, sponsor_likely}, or level is senior+, or ≥5 years required, or students-only phrases, or a Master's/PhD is required (a sentence that has a high-degree term plus a "must" term and no bachelor's or soft term).

**H-1B sponsor:** ≥5 filings under a matching employer name with at least one in the last 3 years.

**Letter checker:** every number in a letter (normalised as floats, URLs removed) must appear in the CV/projects facts. Also flags dash characters, " - " used as a dash, banned phrases (word-boundary), word count outside the range, and a missing sign-off.

## How it's tested

- **74 tests pass in ~1 s** (I ran them). Files: checker, db, dedupe, http, labels, level, page, pipeline, sources.
- The README says they cover dedupe, location labels, the "never overwrite my status" rule, tracker edits, the writing checker, and a full offline run with sample data.
- **Ranking quality is not measured in the repo yet.** There's no labelled set of "good" jobs, and no link from fit score to interview rate.
- **Label accuracy is not measured** (no labelled sentences).

## Known weaknesses

- **Hand-tuned weights and thresholds** with no validation.
- **Regex-based labelling** misses unusual phrasing and can misfire (e.g. negations beyond the ones handled).
- **MiniLM similarity** is topical, not a judgement on seniority or depth.
- **Company name matching** for registers and H-1B uses cleaned names, so there can be false matches or misses.
- **Serial fetching.** One process, per-host waits. The daily run time grows with sources.
- **Single-user design.** Profile files, a local DB, and personal data in the repo.
- **Letter drafting is outside the code**, so its quality depends on the chat session. Only mechanical checks are automated.
