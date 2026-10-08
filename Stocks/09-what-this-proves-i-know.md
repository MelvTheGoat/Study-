# Stocks: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Stocks

---

## 1. Reading company accounts

**Simple explanation:** the three main statements (income, balance sheet, cash flow) and how the key figures come out of them.

**In this project:** 29 figures worked out from SEC filings and Nigerian results PDFs, from margins and cash conversion to interest cover and cash runway.

**Also be ready to explain:**
- **Profit vs cash flow**, and why a profitable company can still run out of cash.
- **Free cash flow** = operating cash flow minus spending on equipment.
- **Net debt** and **interest cover**.
- **Why banks are different**: borrowing and lending is their business, so debt and margins mean something else (the tool explains them differently).
- **Dilution**: new shares shrink every existing holder's slice.

---

## 2. Valuation basics

**Simple explanation:** what you pay for a company compared with what it earns or owns.

**In this project:** P/E from market cap ÷ last twelve months' profit, price-to-book, dividend yield, five years of P/E history, and comparison against peer medians.

**Also be ready to explain:**
- **Why a P/E needs a profit** (no P/E for a loss).
- **Relative valuation**: against the company's own past and against peers.
- **Value traps**: cheap because the market has seen a problem.
- **Median vs mean** for peer groups, and the interquartile range.

---

## 3. Financial data engineering

**Simple explanation:** turning raw official data into clean, dated, trustworthy figures.

**In this project:** SEC company facts parsed to one value per period (latest filing wins), labels chosen per period, TTM and single quarters derived, prices stored as Parquet and read with DuckDB with merge-on-key writes.

**Also be ready to explain:**
- **XBRL** tags and why companies use different ones.
- **Restatements** and point-in-time data.
- **Idempotent writes**: running a job twice gives the same result as once.
- **Split and dividend adjustment**, and why a bonus issue is treated like a split.

---

## 4. Working with rate-limited APIs

**Simple explanation:** staying inside what a free service allows, and catching its quirks.

**In this project:** the SEC's 10-a-second limit and contact-email header; Twelve Data's 800 a day and 8 a minute, handled with incremental fetching and a 700-request nightly budget.

**Also be ready to explain:**
- **Budgets and carry-over** to the next run.
- **Silent wrong answers** from APIs: the inverted split field, the exclusive end date.
- **Validating a response** before trusting it (the brotli near miss).

---

## 5. Data provenance and honest UX

**Simple explanation:** showing users where each number came from, how old it is, and what's missing.

**In this project:** the `Figure` type with sources, as-of dates, notes, a stale flag and a `missing` sentence; amber for stale data; two-tier pages (sentences first, detail on tap).

**Also be ready to explain:**
- **Progressive disclosure** in interface design.
- **Why "unknown" must never look like "fine"** (checks that can't run aren't counted as passed).
- **Accessible charts**: the same numbers in a table.

---

## 6. Rule-based checks

**Simple explanation:** fixed, written rules that flag warning signs.

**In this project:** eight checks (price fall, sales trend, profitability, cash burn, debt payments, new shares, ease of selling, value against peers), each showing its rule and status (ok, watch, concern, info, unknown).

**Also be ready to explain:**
- **Rules vs models**: rules are explainable and arguable, but hand-tuned.
- **Thresholds** and why each one is shown to the user.

---

## 7. Data licensing, terms and ethics

**Simple explanation:** checking you're allowed to collect and show data before you do.

**In this project:** `DATA_SOURCES.md` records every source with dates and terms. NGX forbids automated collection, so PDFs are uploaded by a person and deleted after reading. US prices are licensed for personal use, so the site is private and publishing is gated.

**Also be ready to explain:**
- **robots.txt vs terms of use**: they aren't the same thing.
- **Personal use vs redistribution.**
- **Authorised data vendors** and why the source of resold data matters.

---

## 8. Document extraction with a person in the loop

**Simple explanation:** reading numbers out of PDFs, then having a person confirm them.

**In this project:** `ngx/pdf.py` finds the three statements by their content, picks the year-to-date column by matching profit before tax, checks the balance sheet balances, and writes a draft with page references, marked unchecked.

**Also be ready to explain:**
- **Text PDFs vs scanned PDFs** (and when OCR would be needed).
- **Self-consistency checks** as a cheap form of validation.

---

## 9. Static sites and serverless publishing

**Simple explanation:** building all pages in advance as plain files, then hosting them.

**In this project:** Jinja templates rendered nightly, SVG charts at build time, Cloudflare Pages behind Cloudflare Access.

**Also be ready to explain:**
- **Static vs dynamic sites**, and when each fits.
- **Zero-trust access** with email one-time codes.
- **Progressive enhancement**: pages work without JavaScript.

---

## 10. CI/CD and scheduled jobs

**Simple explanation:** automatic checks on every push, and jobs that run on a timetable.

**In this project:** a nightly workflow (weekdays, 01:17 UTC) with the Actions cache as storage and a publish safety switch; a PDF inbox workflow; a workflow that fetches SEC fixtures on a runner; tests run twice, once with sockets disabled.

**Also be ready to explain:** cron in GitHub Actions, cache keys and restore keys, secrets vs variables, concurrency groups, and why scheduled jobs pause after 60 days without activity.
