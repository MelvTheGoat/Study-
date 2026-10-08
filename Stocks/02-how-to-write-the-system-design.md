# Stocks: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Stocks

Use this on a whiteboard or in an interview. The system is a **nightly batch job that builds a static, private website**. There's no server answering requests.

---

## Step 1: Requirements (2–3 min)

Say the problem in one line:
> "A private site that helps one person understand a US or Nigerian company: sourced figures, plain explanations and warning checks, never buy or sell advice."

**Functional**
1. A page per company: plain sentences first, details on tap.
2. Every figure shows its source and date, or a sentence saying why it's missing.
3. Eight warning checks, each with its rule written out.
4. P/E against the company's own five-year history and against similar companies.
5. What changed in the latest report.
6. A watchlist, a compare page (up to four) and a data-freshness page.
7. Phone alerts when a watched company changes.
8. Nigerian companies from their own results PDFs.

**Non-functional**
- **Trustworthy:** nothing guessed, stale data marked.
- **Free:** every service on a free plan.
- **Private:** behind a login, because the price data is for personal use.
- **Legal data only:** no scraping a site whose terms forbid it.
- **Hands-off:** rebuilds itself every weekday night.

---

## Step 2: Numbers and scale (1–2 min)

| Thing | Number | Source |
|---|---|---|
| Companies today | 3 watched + 18 tracked (US); 0 Nigerian | `watchlist.yaml` |
| Figures per company | 29 | `metrics/core.py` |
| Warning checks | 8 | `metrics/checks.py` |
| SEC limit | 10 requests a second, contact email required | `sec_client.py` |
| Twelve Data free plan | 800 a day, 8 a minute; budget 700 | `sync.py` |
| Stale after | price 4 days, accounts 140 days | `company.py` |
| P/E history | quarter ends over 5 years | `history.py` |
| Build | weekdays, 01:17 UTC | `nightly.yml` |
| Tests | 436 pass, 1 skipped | my run |

**Say:** "Scale is tiny. The constraints are free-plan rate limits and data licences, not traffic."

---

## Step 3: High-level boxes (3 min)

```
[watchlist.yaml] --+
[SEC] -------------+--> [Sync (cache)] --> [Assemble CompanyData] --> [Metrics + checks] --> [Explain] --> [Render HTML] --> [Cloudflare Pages + login]
[Twelve Data] -----+                              ^                         |
[PDF upload] --> [PDF reader: draft + 2 checks] --+                         +--> [Alerts: tonight vs last night] --> [ntfy]
```

---

## Step 4: Deep dive on each part (8–10 min)

### 4a. The Figure type
- Value, unit, currency, as-of date, sources, notes, stale flag.
- Or a `missing` sentence shown in place of the number.
- A test checks every figure for every test company has one or the other.

### 4b. US accounts from the SEC
- One "company facts" JSON per company. Only 10-K/10-Q. Latest filing wins per period (restatements).
- Each figure maps to several SEC labels, chosen **per period** (Apple's revenue label changed in 2018).
- Last twelve months = last full year + year-to-date − same part of last year. Q4 = year − nine months.

### 4c. US prices from Twelve Data
- Incremental: only days not held yet. Stop at 700; carry on tomorrow.
- Three traps: the split `ratio` field is inverted (use `from_factor / to_factor`, checked against the text), `end_date` is exclusive, and there's no adjusted close (adjust it yourself).
- Never store today's unfinished bar.

### 4d. Figures and checks
- 29 figures, one function each. No P/E for a loss; no price-to-book for negative equity.
- 8 checks with written rules, e.g. "Concern if operating profit covers interest less than 1.5 times; watch under 3."

### 4e. Context
- P/E history: five years of quarter ends, the middle half, and where today sits.
- Peers: by SIC code, widening 4 → 3 → 2 digits. US peer search cuts the SEC industry list to major-exchange companies and ranks by revenue closeness.

### 4f. Explanations
- Hand-written glossary: what it is, what to compare it with, how it misleads.
- Summary sentences built only from the company's own figures.
- A test bans advice words ("buy", "undervalued", "you should").

### 4g. Nigeria
- NGX terms forbid automated collection, so prices are typed in.
- Upload a results PDF → a workflow drafts the figures with page numbers → two checks (profit before tax matches; balance sheet balances) → marked unchecked until a person confirms → PDF deleted.

### 4h. Publishing and alerts
- Static HTML on Cloudflare Pages, behind Cloudflare Access (email one-time PIN).
- A safety switch: no publishing until `SITE_IS_PRIVATE` is set.
- Alerts compare snapshots and send one ntfy message.

---

## Step 5: Bottlenecks (2 min)

1. **Twelve Data's free plan:** 8 a minute. Handled by incremental fetching and a nightly budget.
2. **Lost cache:** the next run re-fetches and catches up over a night or two.
3. **NGX data:** no free, allowed feed. Handled by PDFs and hand-typed prices.
4. **SEC label drift:** handled by per-period label lists, each added because a real filing needed it.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Static site built nightly | A live web app | Free hosting, nothing to keep running | Data is up to a day old |
| Hand-written explanations | AI-generated text | Can be checked and tested | Must be written for every figure |
| Show "why missing" | Fill in a guess | Trust | Some pages have gaps |
| PDF upload + hand prices for Nigeria | Scraping NGX | Its terms forbid it | Manual work |
| Free Twelve Data plan | Paid feed | Free | Slow first run, 800 a day |
| Dropping the AI agent | Finishing it | A tool the owner uses every day | The earlier work was removed |

Close with: "Next is adding Nigerian companies and deciding on a licensed Nigerian price feed."
