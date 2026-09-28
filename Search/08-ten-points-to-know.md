# Search (Job Hunt): 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Search (private)

1. **It finds ML/AI jobs, labels whether you can get them from Nigeria, ranks them against your CV, and never applies for you.**
   *Why it matters:* reachability is the core idea, not just matching.

2. **Sources: 405 company boards on 9 ATSs, plus remote boards, aggregators, Amazon (Africa) and HN "Who is hiring?". No LinkedIn or Indeed.**
   *Why it matters:* official, permitted sources only.

3. **Polite fetching: User-Agent, per-host waits, retries with backoff, cached aggregators. One broken source never stops the run.**
   *Why it matters:* reliability across hundreds of sources.

4. **Dedupe on apply URL or company+title+location. The longer description wins.**
   *Why it matters:* the same job appears in many places.

5. **Labels: remote_open, nigeria, africa, sponsor_yes, sponsor_likely, sponsor_unknown, restricted, with the exact sentence as evidence.**
   *Why it matters:* checkable rules you can trust.

6. **Sponsor evidence: UK and NL registers (weekly) and US H-1B filings (≥5, one in the last 3 years).**
   *Why it matters:* catches sponsors that don't say so in the post.

7. **Fit = 100 × (0.30 CV + 0.20 project + 0.20 skills + 0.22 role + 0.08 domain) × level multiplier.**
   *Why it matters:* know the formula exactly.

8. **Embeddings: all-MiniLM-L6-v2 on CPU, cached in SQLite, with a word-hash fallback.**
   *Why it matters:* free, private and fast. No AI API calls for scoring.

9. **Re-runs never overwrite status, notes, date applied, letter file or date found. Spreadsheet edits are read back.**
   *Why it matters:* a daily tool must never lose your work.

10. **The letter checker flags any number not in your CV, plus dashes, banned phrases, length and sign-off. 74 tests pass. Score quality is not measured yet.**
    *Why it matters:* shows careful use of AI, and honesty about what's unvalidated.
