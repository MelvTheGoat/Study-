# Search (Job Hunt): What This Proves I Know

Repo: https://github.com/MelvTheGoat/Search (private)

---

## 1. Semantic search with embeddings

**Simple explanation:** turn text into vectors so that texts with similar meaning are close together, then compare them with cosine similarity.

**In this project:** MiniLM embeddings of job chunks, CV and projects, mean-pooled, cosine, mapped to 0–1.

**Also be ready to explain:** sentence-transformers vs word vectors, chunking long text, normalisation, cosine vs dot product, vector databases (FAISS, pgvector), and bi-encoders vs cross-encoders (re-ranking).

---

## 2. Ranking and scoring design

**Simple explanation:** combine several signals into one number to sort items.

**In this project:** weighted average of five signals times a level multiplier, a skills prior, and a "why" and gaps per job.

**Also be ready to explain:** learning-to-rank, how to evaluate a ranking (precision@k, NDCG, rank correlation), why you normalise features, and cold-start without labels.

---

## 3. Information extraction with rules

**Simple explanation:** pull facts from text with patterns.

**In this project:** regexes for seniority, years of experience, sponsorship, restrictions, degrees and student-only posts, all sentence-level with evidence kept.

**Also be ready to explain:** precision vs recall of rules, negation handling, when to move to a classifier (weak supervision, labelled sentences), and named-entity recognition for locations.

---

## 4. Data pipelines and ETL

**Simple explanation:** extract from sources, transform into one shape, load into storage.

**In this project:** fetch → normalise to one `Job` model → dedupe → filter → label → score → upsert → export.

**Also be ready to explain:** idempotent loads, upserts, failure isolation, incremental runs, and scheduling (cron, Task Scheduler).

---

## 5. Entity resolution (dedupe)

**Simple explanation:** decide when two records are the same thing.

**In this project:** apply-URL key plus a company/title/location hash. Keep the richer record.

**Also be ready to explain:** fuzzy matching, blocking, name normalisation (legal suffixes like "Inc", "Ltd"), and the trade-offs of false merges vs missed duplicates.

---

## 6. Web APIs and responsible scraping

**Simple explanation:** read data from websites in ways they allow.

**In this project:** public ATS APIs, RSS, rate limits, caching, a clear User-Agent, a board health check, and respecting sites that forbid scraping.

**Also be ready to explain:** robots.txt, terms of service, HTTP status codes, backoff, and pagination.

---

## 7. Data modelling for user-owned state

**Simple explanation:** separate data the system owns from data the user owns, and never let the system overwrite the user's.

**In this project:** `USER_FIELDS` untouched by upsert. Spreadsheet and page edits synced back.

**Also be ready to explain:** conflict resolution (last-write-wins vs field ownership), and sync between systems.

---

## 8. Using LLMs safely

**Simple explanation:** AI can invent facts, so check its output with plain code.

**In this project:** the letter checker verifies every number against the CV facts, bans phrases and enforces length.

**Also be ready to explain:** hallucination, grounding, guardrails, human-in-the-loop, and why "facts only from a source file" is a strong rule.

---

## 9. Immigration and work-rights signals (domain knowledge)

**Simple explanation:** whether you can legally take a job depends on the country and the employer.

**In this project:** UK licensed sponsors, the Dutch IND recognised sponsors, US H-1B history, ECOWAS free movement.

**Also be ready to explain:** what a sponsor licence is, why a register entry isn't a guarantee, and remote-within-country restrictions.

---

## 10. Document generation

**Simple explanation:** produce files (Word/PDF) from structured data.

**In this project:** tailored CVs from `cv_data.yaml` via python-docx: same facts, with order and headline chosen per job, and a layout that applicant tracking systems can read.

**Also be ready to explain:** templating, and why simple one-column layouts parse better in applicant tracking systems.
