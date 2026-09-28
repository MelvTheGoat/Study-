# Search (Job Hunt): Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Search (private)

---

## 60-second pitch

> "I built a daily job-search pipeline for ML roles, designed around one question most job sites skip: can I actually get this job from Nigeria? It pulls from 405 company career boards on nine applicant tracking systems plus several job boards, only sources that allow it. It dedupes, filters for data and AI roles, and labels each job: remote and open to Africa, in Nigeria, elsewhere in Africa, abroad with a likely visa sponsor, or restricted. For sponsors it uses the UK and Dutch sponsor registers and US H-1B history, and it saves the exact sentence behind each label.
>
> Then it scores fit from 0 to 100 on my laptop, using a small embedding model plus rules for skills, role and seniority. Everything lands in SQLite and a spreadsheet, and re-runs never overwrite my notes. It never applies for me. It checks drafted cover letters so no letter contains a number that isn't in my CV. 74 tests."

---

## Questions and honest answers

### 1. "Why not just use LinkedIn?"
LinkedIn and Indeed don't allow scraping, and their filters don't handle work rights well. Company ATS boards are the original source and have public APIs. I use those plus boards that allow automated reading.

### 2. "How does the fit score work?"
A weighted average of five parts: CV similarity 0.30, best-project similarity 0.20, skills overlap 0.20, role type 0.22 and domain 0.08. That's multiplied by a level score (1.0 for junior, 0.8 for mid, about 0.35–0.45 for senior and above), times 100. Similarities are MiniLM cosine scores mapped from 0.30–0.70 onto 0–1.

### 3. "How do you know the score is any good?"
Honestly, I don't yet. The weights are hand-set. It isn't validated against outcomes. To measure it, I'd log applications and interviews and check whether a higher score predicts interviews (e.g. rank correlation or precision at top-k). Until then it's a sorting aid, and every score comes with a "why" and skill gaps.

### 4. "Why embeddings and not an LLM for scoring?"
Free, private and fast. MiniLM runs on a CPU and results are cached. An LLM would be slower, cost money or quota, and send my CV to a third party. The trade-off is shallower matching.

### 5. "How do you detect visa sponsorship?"
Three signals: phrases in the post (both "we sponsor" and "no sponsorship"), the UK and Netherlands sponsor registers (refreshed weekly), and US H-1B history (at least 5 filings, one in the last 3 years). The post itself wins over the registers. I keep the evidence so I can check it.

### 6. "What about false labels?"
They happen. It's regex over sentences. That's why the exact restricting sentence is saved, restricted jobs get their own tab, and nothing is deleted. A small classifier trained on labelled sentences would be the next step.

### 7. "How do you handle duplicates?"
Two keys: the apply URL, and a hash of company + title + location. The copy with the longer description wins, so the full company post beats an aggregator snippet.

### 8. "What happens when a source breaks?"
Each source is wrapped. A failure adds an error to the run summary and the run carries on. `verify-companies` tests every board, tries other ATSs for dead tokens, and moves dead boards out of the list.

### 9. "How do you make re-runs safe?"
The upsert never touches user-owned fields: status, notes, date applied, letter file and date found. Before rewriting the spreadsheet, it reads my edits back from it. There's a test for that rule.

### 10. "Isn't using AI for cover letters risky?"
Yes, which is why the code checks them. Any number in a letter that doesn't appear in my CV or projects file is flagged, along with banned phrases, dashes, length and sign-off. It catches invented results. It doesn't judge tone, so I still read every letter.

### 11. "How would you scale it to many users?"
Async fetching with per-host limits, Postgres with per-user data, a vector index for embeddings, a task queue for fetch, score and CV builds, encrypted personal data, and learned ranking weights once there's outcome data.

### 12. "What would break first?"
Sources changing their APIs or HTML. That's the most frequent failure. Then run time, because fetching is serial with polite waits.

### 13. "Why SQLite and Excel?"
It's one person's tool. SQLite is zero-setup, and Excel is where I actually edit status and notes. The two-way sync keeps them consistent.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Unvalidated score | "Is 80 better than 70?" | "It's a sorting aid. I haven't measured it against interviews yet. Here's how I would." |
| Regex labels | "Regex for legal text?" | "It's transparent, and evidence is kept for every label. A classifier is the upgrade." |
| Letters outside the code | "So an AI writes your letters?" | "An AI assistant drafts them. Code checks every number against my CV, and I read and edit each one." |
| Single user | "It's not a product." | "Right. It's a personal tool. I described how I'd make it multi-user." |
| Personal data in repo | "Your CV is in git?" | "The repo is private for that reason. At scale I'd separate and encrypt it." |
