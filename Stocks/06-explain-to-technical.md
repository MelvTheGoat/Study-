# Stocks: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Stocks

---

## Summary

A personal research tool for US and NGX-listed companies: a nightly batch job that syncs data, computes sourced figures and rule-based checks, and renders a static, private site. It never gives buy/sell advice. **436 tests pass, 1 skipped (~12 s, my run).** Runtime dependencies: pyyaml, httpx, duckdb, pyarrow, jinja2, pypdf.

History: started 18 Sep 2026 as an eval-first LLM agent (agent loop, BM25 baseline, graders, LLM judge with a kappa gate, Kaggle/vLLM runner, trace viewer). On 6 Oct 2026 those were removed, the package was renamed from `stockagent` to `research`, and the data layer was kept. Merged to `main` on 8 Oct (PR #3). 79 commits.

## Architecture

```
src/research/
  company.py               Figure (value/unit/currency/as_of/sources/missing/notes/stale), CompanyData
  data/                    models, Parquet + DuckDB store (merge on natural key), returns (split/bonus adj.),
                           Twelve Data client + parser, Alpha Vantage parser, calendar, crosscheck
  fundamentals/
    concepts.py            figure -> ordered SEC us-gaap labels, chosen per period
    sec_client.py          User-Agent with contact email, < 10 req/s
    sec.py                 company facts -> one Fact per figure per period (latest filing wins)
    financials.py          TTM, single quarters (Q4 = FY - 9M), balances on a date, debt from parts
  metrics/
    core.py                29 figures, CompanyData -> Figure
    checks.py              8 checks, each with a written rule
    history.py             P/E at quarter ends over 5 years, interquartile band
    peers.py               SIC peers, widening 4 -> 3 -> 2 digits; peer medians
    changes.py             latest quarter vs same quarter a year earlier
  explain/
    glossary.py            hand-written what / compare / careful per figure
    summary.py             tier-1 sentences built from figures
  ngx/
    manual.py              validated hand-entered figures, source required
    pdf.py                 statement detection, YTD column via PBT match, balance check, draft
  pipeline/
    watchlist.py           watch / track / peers, named errors
    sync.py                SEC nightly; Twelve Data incremental under a 700-request budget
    peer_search.py         SEC SIC listing -> major-exchange filter -> revenue closeness via frames
    assemble.py            builds CompanyData from disk
    alerts.py              snapshot diff -> one ntfy message
  site/
    view.py, render.py     all decisions in view.py; templates only lay out
    charts.py              SVG at build time, points in markup for hover
    templates/, static/    Jinja templates, CSS (light/dark), one small JS file
scripts/build.py           sync -> assemble -> build -> diff -> alerts (--offline to skip fetch)
scripts/read_ngx_inbox.py  PDFs in ngx/inbox/ -> ngx/extracted/TICKER.yaml
.github/workflows/         nightly, ngx-inbox, sec-samples, tests
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| Figure carries provenance | Sources + as-of date, or a `missing` sentence; a test enforces one or the other for every fixture company | Every number is checkable |
| No guessing | No P/E for a loss; no P/B for negative equity; zero dividend yield only if prices and the cash flow statement agree; unrunnable checks aren't counted as passed | Wrong is worse than absent |
| Staleness | Price > 4 days, accounts > 140 days (when a newer report should be out); never store today's unfinished bar | Old data is shown, not hidden |
| Latest filing wins per period | 10-K/10-Q and amendments only | Picks up restatements |
| Per-period concept choice | First label with a value, per period | Apple's revenue label changed in 2018 |
| Derived periods | TTM = FY + YTD − prior YTD; Q4 = FY − 9M; cash flow quarters from YTD differences | Filings don't report these directly |
| P/E from totals | Market cap / TTM net income | Avoids per-share restatement after splits |
| Twelve Data guards | Split factor from `from_factor / to_factor`, checked against the description; exclusive `end_date`; no adjusted close, so adjust ourselves | Each mistake gives silent wrong numbers |
| Rate budgets | SEC 10/s nightly; Twelve Data 700 of 800/day, 8/min, incremental; actions weekly | Free plans |
| Hand-written explanations | Tests: every figure covered, short sentences, regex bans `buy`, `bargain`, `undervalued`, `overvalued`, `opportunity`, `you should` | Fixed text is testable |
| Static site | Jinja → HTML, SVG charts + tables, `<details>` for tier 2, works without JS | Free hosting, nothing running |
| Private by default | Cloudflare Access (email OTP); publish gated on `SITE_IS_PRIVATE=yes` | US prices licensed for personal use |
| NGX via PDFs | Person uploads; reader drafts with page refs, `checked: false`, 2 cross-checks; PDF deleted | NGX terms forbid automated collection and republication |

## Models and algorithms

There's no machine learning. The "algorithms" are accounting arithmetic and rules:

- **Period arithmetic:** TTM, single quarters, balances on a date, debt summed from its parts on one balance-sheet date.
- **29 figures:** price, market cap, P/E, P/B, dividend yield, revenue and profit growth (1y and 3y), net and operating margin, ROE, free and operating cash flow, cash conversion, cash, revenue, net income, debt, net debt, debt/equity, interest cover, current ratio, cash runway, 3-year share change, average daily traded value, no-trade days, 1y and 3y price change, fall from high.
- **Checks:** fixed thresholds, e.g. interest cover < 1.5 concern, < 3 watch; cash runway < 18 months concern, < 3 years watch; share count +20% in 3 years concern, +5% watch.
- **P/E history:** quarter-end close × shares then ÷ TTM profit to that quarter end; median and middle half of the range.
- **Peer search:** SEC company list for the SIC code → listed on a major exchange → ranked by revenue closeness using one frames download.
- **PDF reading:** find the three statements by content; choose the year-to-date column where income-statement PBT equals cash-flow PBT; check assets = liabilities + equity.

## How it's tested

- **436 pass, 1 skipped** (I ran them). Areas: SEC parsing, financials, metrics, checks/peers/history, explanations, site, sync, build end to end, alerts, watchlist, peer search, NGX manual and PDF, store, returns, Twelve Data parser and client, calendar, crosscheck.
- **Real filings, made-up prices.** SEC fixtures are real company facts for Apple, Coca-Cola, JPMorgan, GoPro and McDonald's, trimmed to the labels used. Prices are simple (a flat $100), and comments show the arithmetic from the filing.
- **CI (`tests.yml`):** ruff → pytest → pytest with `--disable-socket`.
- **Fixtures fetched in CI:** the SEC refuses the dev container, so `sec-samples.yml` fetches on a runner and pushes to a `sec-samples` branch.
- **PDF reader on real data:** one report (NGX Group Q1 2026), all main figures correct, both checks passed.

## Known weaknesses

- **No NGX companies entered**, and NGX prices are manual until a licensed feed is chosen (NGX's own API is N125,000 a year for end-of-day prices).
- **PDF reader proven on one real report.** Scanned PDFs can't be read.
- **Cache as storage.** Data lives in the Actions cache; losing it means re-fetching over a night or two.
- **Slow first run** at 8 requests a minute (up to an hour).
- **Deployment status not visible in the repo.** `DEPLOY.md` is a setup checklist.
- **Dead code from the agent era:** `calendar.py`, `crosscheck.py` and the Alpha Vantage parser are tested but unused; some docstrings still mention the eval and agent.
- **GitHub pauses scheduled workflows** after 60 days without repo activity.
