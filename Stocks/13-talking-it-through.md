# Stock Research: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to where it is today. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. When most people look at a share, what do they look at? The price chart. Up or down.

But the chart can't tell you if the company makes money, if it's running out of cash, or if it's drowning in debt.

Those answers are in the company's reports. The trouble is, they're long, and written in a language most people never learned.

So here's our project. A private website, just for me, where I look up a company, American or Nigerian. It shows me the numbers that matter. And next to every number, it explains what it is, what to compare it with, and how it usually misleads.

One rule above all: **it never tells me to buy or sell.** If I want to decide, it lays out the inputs. The decision stays mine, because that's how I learn to make it.

## So what do we need?

Let's list it out:

1. **Data I'm allowed to use**, for both countries.
2. **Share prices**, adjusted properly for splits.
3. **Company accounts**, in the time periods people actually use.
4. **A way for every number to carry its evidence**: where it came from, and when.
5. **The figures themselves**, like profit margin and debt.
6. **Warning checks**, and context: the company's own past, and similar companies.
7. **Explanations**, written by hand, with no advice in them.
8. **A website**, private, cheap, and simple.
9. **A robot** that rebuilds it every night and tells me what changed.
10. **A way in for Nigerian companies**, without breaking any rules.

## A quick word on how this started

Before we start, you should know this repo had a first life. It began on 18 September as a completely different idea: an AI helper that answers questions about stocks, with a test to measure how often it's right.

A lot got built for that: a question set, graders, a simple search baseline, a GPU runner. Then, on 6 October, I changed direction. What I actually wanted was to understand companies myself, every day, not to score a chatbot.

So the AI parts were removed, and the main code folder was renamed to `research`. But the data parts were good, so we kept them. You'll see which ones as we go.

## Step zero: set up the workshop

The basics first. `README.md` is the front page: what the site does, where the numbers come from, and what it costs. Which is nothing, as set up.

`.gitignore` tells git what not to save. That matters more than usual here, because the downloaded data is licensed, and must never land in the repository.

`requirements.txt` lists six tools: PyYAML, httpx, DuckDB, pyarrow, Jinja2 and pypdf. `requirements-dev.txt` adds testing and tidying tools. `pyproject.toml` describes the package, called `research`, and needs Python 3.11 or newer.

The code lives in `src/research/`. And `tests/test_package.py` just checks the package imports and the tests are wired up right.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: what git should never save, including licensed data.
> - `requirements.txt`: the six tools the code needs.
> - `requirements-dev.txt`: extra tools for testing and tidying.
> - `pyproject.toml`: the package details and tool settings.
> - `src/research/__init__.py`: marks the main code folder as a package.
> - `tests/test_package.py`: checks the package and test setup.

## Step one: are we even allowed?

Before we write a single line that collects data, we write down the rules. That's `DATA_SOURCES.md`. Its rule is simple: nothing gets collected until it has an entry here, with the date it was checked.

The big one is the Nigerian Exchange, NGX. Its terms of use say, and I'm quoting, you shall not conduct "any systematic or automated data collection" without their written consent. They also forbid republishing their data. And their licensed data feed costs money: end-of-day prices are N125,000 a year.

So that's a clear no. We don't scrape it. Ever.

And here's a story worth knowing. The NGX site sits behind a firewall that blocked my development machine. So the first check of its terms ran on a GitHub computer instead. And it reported: no rules found about automated access.

That was **wrong**, and it was nearly believed. The page came back compressed in a format that computer couldn't unpack, so it was really just binary noise. And of course a word search over noise finds nothing.

So we fixed it to only ask for formats it can always unpack, and to check the page looks like text before trusting it. The rules were right there on the first try. If we'd believed the first run, we'd have written down permission that doesn't exist.

For the US, it's easier. The SEC, the US regulator, publishes every company's accounts for free. It only asks that you say who you are, with a contact email, and stay under ten requests a second.

> **📁 Files we just created**
> - `DATA_SOURCES.md`: every data source, its terms, its cost, and the date it was checked.

## Step two: one shape for market data

Now, we kept this part from the first life of the project. Two markets, two currencies, naira and dollars, have to live in one database without getting mixed up.

`src/research/data/models.py` says what a price, a dividend or a split looks like. And every record carries its market, its currency and its source. So a naira price can never be mistaken for a dollar one.

`src/research/data/store.py` is the database. The data sits in Parquet files, which are compact tables on disk, and DuckDB reads them with normal database questions. There's no database server to run at all.

And writes are merges, not additions. If you save the same day's price twice, you still have it once. That matters, because a nightly job will often go over days it's seen before.

`src/research/data/returns.py` does the arithmetic for splits. When a company splits each share into four, the price drops to a quarter overnight. Without adjusting, that looks like a crash. And for Nigerian shares, a bonus issue counts as a split too.

There are two more files from before: `calendar.py`, which works out trading days from the data, and `crosscheck.py`, which compares two price sources. They're tested, but honestly, the website doesn't use them yet.

> **📁 Files we just created**
> - `src/research/data/__init__.py`: marks the data folder.
> - `src/research/data/models.py`: what a price, dividend and split look like, always with market and currency.
> - `src/research/data/store.py`: Parquet files plus DuckDB, with merging writes.
> - `src/research/data/returns.py`: split and bonus-issue adjustment.
> - `src/research/data/calendar.py`: trading days, worked out from the data (not used by the site yet).
> - `src/research/data/crosscheck.py`: compares two price sources (not used by the site yet).
> - `tests/test_store.py`, `tests/test_returns.py`, `tests/test_calendar.py`, `tests/test_crosscheck.py`: their tests.

## Step three: share prices, and three traps

US prices come from Twelve Data, a price service with a free plan. Free means limits: 800 requests a day, 8 a minute.

`src/research/data/sources/twelvedata.py` reads its answers. And this file guards three traps. Each one gives you confident wrong numbers, not an error, which is the worst kind.

Trap one. A split arrives with a field called `ratio`, and it's the wrong way round. A four-for-one split says 0.25. If you read that, every split-adjusted return is wrong, by a factor of sixteen for Apple.

So we take the factor from two other fields, and check it against the words in the description. If they disagree, it stops rather than guesses.

Trap two. When you ask for prices up to a date, that last date isn't included. Get that wrong and a stock mysteriously stops a day early.

Trap three. There's no "adjusted" price. So we do the adjusting ourselves, with `returns.py` from the last step.

`twelvedata_client.py` does the actual fetching. It counts its requests and waits before going over 8 a minute.

Why? Because going over doesn't fail loudly. You get an answer that, read carelessly, looks like a stock with no data.

There's also `alphavantage.py`, a reader for another free price service. It's tested, but not used today.

For the tests, we keep sample answers in `data/samples/`. And here's a nice detail: the Twelve Data samples have the right shape, but the prices are made up. Their terms don't allow their data to be republished, so real figures don't belong in the repository.

> **📁 Files we just created**
> - `src/research/data/sources/__init__.py`: marks the price sources folder.
> - `src/research/data/sources/twelvedata.py`: reads Twelve Data answers, with the three traps guarded.
> - `src/research/data/sources/twelvedata_client.py`: fetches within the free limit.
> - `src/research/data/sources/alphavantage.py`: a reader for another price service (not used today).
> - `data/samples/twelvedata/`: sample answers with made-up prices, plus a `README.md` saying so.
> - `data/samples/alphavantage/`: sample answers for the other service.
> - `tests/test_twelvedata.py`, `tests/test_twelvedata_client.py`, `tests/test_alphavantage.py`: their tests.

## Step four: company accounts from the SEC

Now the accounts. The SEC publishes one file per company, called "company facts". It has every number the company ever tagged in its filings.

`src/research/fundamentals/sec_client.py` fetches it the way the SEC asks: with a contact email, and under ten requests a second. Without the email, you just get refused.

`src/research/fundamentals/sec.py` reads that file. But the same number shows up many times: in the original report, again as "last year" in the next report, and again if it's corrected. So for each period, the most recently filed value wins. That way, corrections are picked up.

Then, labels. Companies tag the same line with different labels, and change them over the years. Apple's revenue label changed in 2018. So `src/research/fundamentals/concepts.py` keeps, for each figure, a list of labels, best first.

The choice is made separately for each period. And every label in there is there because a real filing needed it.

Next, the periods people actually want. Filings don't hand them over. `src/research/fundamentals/financials.py` works them out. Here's the main one, "the last twelve months":

> last full year, plus this year so far, minus the same part of last year.

And the fourth quarter? Nobody reports it on its own. So it's the full year minus the first nine months.

Now, testing this. We want real filings, but the SEC refuses requests from my development machine. So `scripts/fetch_sec_samples.py` runs on a GitHub computer instead, through `.github/workflows/sec-samples.yml`, and pushes the files to a separate branch.

Those files are big. So `scripts/trim_sec_fixtures.py` cuts them down to just the labels we use. They end up in `tests/fixtures/sec/`: real filings for Apple, Coca-Cola, JPMorgan, GoPro and McDonald's. `tests/conftest.py` has small helpers to load them.

`tests/test_fundamentals.py` checks the readings. And wherever a test expects a number, a comment shows the sum from the filing, so you can check it by hand.

> **📁 Files we just created**
> - `src/research/fundamentals/__init__.py`: marks the accounts folder.
> - `src/research/fundamentals/sec_client.py`: fetches from the SEC, politely.
> - `src/research/fundamentals/sec.py`: one value per figure per period, latest filing wins.
> - `src/research/fundamentals/concepts.py`: which SEC labels feed each figure.
> - `src/research/fundamentals/financials.py`: last twelve months, single quarters, balances on a date.
> - `scripts/fetch_sec_samples.py` and `.github/workflows/sec-samples.yml`: fetch real SEC files on a GitHub computer.
> - `scripts/trim_sec_fixtures.py`: cuts them down for the tests.
> - `tests/fixtures/sec/`: real filings for five companies, trimmed.
> - `tests/conftest.py`: helpers that load those filings.
> - `tests/test_fundamentals.py`, `tests/test_sec_client.py`: their tests.

## Step five: every number carries its evidence

This is the heart of the whole project. Let's pause on it.

`src/research/company.py` defines a `Figure`. It's not just a number. Here's the real shape:

```python
class Figure:
    key: str
    value: float | None
    unit: Unit
    currency: Currency | None = None
    as_of: date | None = None
    sources: tuple[str, ...] = ()
    # Why there is no value, in a sentence a reader can act on.
    missing: str | None = None
```

So every figure has its sources and the date it describes. And if there's no value, it has a `missing` sentence instead, and the page shows that sentence. A test checks every figure for every test company has one or the other.

It also has a `stale` flag. A price older than four days goes amber. So do accounts older than 140 days, because by then a newer report should be out.

The same file has `CompanyData`: everything we know about one company, in one place. And `src/research/format.py` decides how numbers are written, so a figure looks the same in a sentence, a table and a check.

> **📁 Files we just created**
> - `src/research/company.py`: the `Figure` with its evidence, and `CompanyData`.
> - `src/research/format.py`: how numbers are written on the page.

## Step six: the figures

Now we work out the numbers. `src/research/metrics/core.py` has 29 figures, one small function each. Price, company value, P/E, margins, cash flow, debt, interest cover, cash runway, price moves, and so on.

And the rule everywhere is: nothing is guessed. Here's the P/E refusing when there's a loss:

```python
f"No P/E: the company made a loss over the twelve months to "
f"{fmt.long_date(profit.end)}. A P/E divides by profit, so it needs a profit.",
```

Same idea elsewhere. No price-to-book when what it owns is less than what it owes. And a dividend yield of zero only if both the price data and the company's own cash flow statement agree nothing was paid.

`tests/test_metrics.py` checks them. The prices in it are made up, a flat 100 dollars, so every expected figure can be worked out by hand.

> **📁 Files we just created**
> - `src/research/metrics/__init__.py`: marks the metrics folder.
> - `src/research/metrics/core.py`: the 29 figures.
> - `tests/test_metrics.py`: checks them, with sums in the comments.

## Step seven: warning checks and context

A share can look cheap for two reasons. Either the market missed something, or the market saw something. `src/research/metrics/checks.py` looks for the second kind, with eight checks.

Is the price down by more than half in three years? Are sales shrinking? Is it losing money? Is it burning cash, and for how long can it?

Can profit cover the interest on its debt? Is it issuing lots of new shares? Could you sell easily? And how is it priced against similar companies?

Each check shows its rule, so I can disagree with it. Like: "Concern if operating profit covers interest less than 1.5 times; watch under 3." And if a check can't run, it says "unknown", never "ok".

Then context, because a number alone means little. `src/research/metrics/history.py` works out the P/E at every quarter end for five years, finds the middle half of that range, and shows where today's sits.

`src/research/metrics/peers.py` compares with similar companies, picked by industry code. If there aren't enough at four digits, it widens to three, then two, and says which it used.

And `src/research/metrics/changes.py` answers the questions everyone asks when a report comes out. Did sales, profit, cash, debt or the share count go up or down, against the same quarter a year earlier?

> **📁 Files we just created**
> - `src/research/metrics/checks.py`: the eight warning checks, each with its rule.
> - `src/research/metrics/history.py`: five years of P/E, and where today sits.
> - `src/research/metrics/peers.py`: comparing with similar companies.
> - `src/research/metrics/changes.py`: what changed in the latest report.
> - `tests/test_analysis.py`: tests for checks, peers, history and changes.

## Step eight: explanations, written by hand

Now the teaching part. `src/research/explain/glossary.py` has, for every figure, three short pieces of hand-written text: what it is, what to compare it with, and how it most often misleads.

Why by hand, not by an AI? Because being wrong about what a figure means is worse than not showing it. Fixed text can be read carefully once, and tested forever.

`src/research/explain/summary.py` writes the plain sentences at the top of each page. How expensive is it, which way is it going, does it make or burn cash, which checks does it trip. Every sentence is built from the company's own figures, so each one traces back to a number on the page.

`tests/test_explain.py` checks every figure has an explanation, the sentences stay short, and nothing ever gives advice. It literally searches for words like "buy", "bargain", "undervalued" and "you should".

> **📁 Files we just created**
> - `src/research/explain/__init__.py`: marks the explanations folder.
> - `src/research/explain/glossary.py`: what / compare / careful, for every figure.
> - `src/research/explain/summary.py`: the plain sentences at the top of each page.
> - `tests/test_explain.py`: complete, plain, and never advice.

## Step nine: the website

Now we build the pages. And we keep them plain: just files, rebuilt each night, with nothing running on a server. That's free to host and can't break in interesting ways.

`src/research/site/view.py` decides everything one page shows, before any web code is written. Every number, sentence and judgement is decided here. That keeps it all testable without reading web pages.

`src/research/site/charts.py` draws small line charts as shapes, right when the site is built. And under every chart, there's a table with the same numbers.

`src/research/site/render.py` writes out the folder of pages, using templates in `src/research/site/templates/`. `base.html` is the shared frame, `index.html` the home page with the watchlist, `company.html` a company page, `compare.html` for up to four side by side, `data.html` for how fresh everything is, and `macros.html` for small reusable pieces.

Each page has two layers. First, a few plain sentences, so you get the picture in thirty seconds. Then, under each heading, the detail opens when you tap it.

`src/research/site/static/style.css` gives it light and dark themes. And `site.js` is one small script for chart hover, search and the compare picker. Every page still works without it.

> **📁 Files we just created**
> - `src/research/site/__init__.py`: marks the website folder.
> - `src/research/site/view.py`: decides everything a page shows.
> - `src/research/site/charts.py`: small charts drawn when the site is built.
> - `src/research/site/render.py`: writes the folder of pages.
> - `src/research/site/templates/`: `base.html`, `index.html`, `company.html`, `compare.html`, `data.html`, `macros.html`.
> - `src/research/site/static/style.css` and `site.js`: looks, and one small script.
> - `tests/test_site.py`: the site, built from real filings and made-up prices.

## Step ten: the nightly robot

Now let's make it run itself. First, which companies? `watchlist.yaml` lists them, and you can edit it right on GitHub.

"Watch" means shown first and checked for changes; right now that's Apple, Coca-Cola and McDonald's. "Track" means built and searchable; that's 18 other big US companies. And you can pick peers by hand if the automatic choice looks wrong.

`src/research/pipeline/watchlist.py` reads it. If there's a typo, it stops and names it. A company quietly dropping off the list is worse than a build that stops.

`src/research/pipeline/sync.py` does the fetching. SEC accounts are free, so they're simply fetched again every night. Prices only for the days we don't have yet, stopping at 700 requests, below the free 800.

If it runs out, the next night carries on. Dividends and splits once a week. And it never saves today's price before the day is finished.

`src/research/pipeline/peer_search.py` finds similar US companies by itself. It takes the SEC's list for the industry, keeps only companies still on a major exchange, and ranks them by how close their sales are. Why the filter? Of the 100 companies filed under computer makers, only six are still trading.

`src/research/pipeline/assemble.py` puts each company together from what's on disk. And `src/research/pipeline/alerts.py` compares tonight with last night. A new report, a check that got worse, a big price move, or data that stopped arriving becomes one short message to my phone, through a free app called ntfy.

`scripts/build.py` runs the whole night: fetch, assemble, build, compare, alert. And `.github/workflows/nightly.yml` runs that every weekday night, at 01:17 UTC, after the US market has closed.

There are two clever bits in that workflow. The data is kept in GitHub's cache between runs, so each night only fetches what's new.

And there's a safety catch. The price data is licensed for personal use, so the site must be private. The workflow won't publish anything until a setting called `SITE_IS_PRIVATE` says the login is in place.

> **📁 Files we just created**
> - `watchlist.yaml`: the companies to watch and track.
> - `src/research/pipeline/__init__.py`: marks the nightly run folder.
> - `src/research/pipeline/watchlist.py`: reads the watchlist, naming any mistake.
> - `src/research/pipeline/sync.py`: fetches accounts and prices within the limits.
> - `src/research/pipeline/peer_search.py`: finds similar US companies.
> - `src/research/pipeline/assemble.py`: puts each company together.
> - `src/research/pipeline/alerts.py`: tonight vs last night, then one phone message.
> - `scripts/build.py`: runs the whole night.
> - `.github/workflows/nightly.yml`: runs it every weekday night, with the safety catch.
> - `tests/test_watchlist.py`, `tests/test_sync.py`, `tests/test_peer_search.py`, `tests/test_build.py`, `tests/test_alerts.py`: their tests.

## Step eleven: Nigerian companies, the allowed way

Remember, we can't collect from the NGX site. So how do Nigerian companies get in?

Through their own reports. A person, me, downloads a company's results PDF in a browser, the normal way, and uploads it into `ngx/inbox/`. That's ordinary use of the site.

That upload starts `.github/workflows/ngx-inbox.yml`, which runs `scripts/read_ngx_inbox.py`. And the reading is done by `src/research/ngx/pdf.py`.

It finds the three main statements by what's in them. Then it picks the right column. Results PDFs put several side by side, like this quarter and the year so far. So it finds profit before tax on the cash flow statement, and picks the income statement column with that same number.

Then it checks its own work two ways: profit before tax must match on both statements, and the balance sheet must balance. It writes everything, with the page each figure came from, into `ngx/extracted/`, marked `checked: false`. Then it deletes the PDF, so nothing from the site is republished.

Until I compare the figures with the PDF and change that to `true`, the company's page says they were read automatically and haven't been checked.

Company details, like the number of shares, and the price, go in `ngx/companies.yaml`. The price is typed in by hand, with its date, and goes amber after four days. `src/research/ngx/manual.py` reads that file, and every figure must have a source.

`ngx/README.md` explains all of this step by step. And `ngx/email-to-ngx.md` is a draft email to NGX, asking about access for personal, non-commercial use.

> **📁 Files we just created**
> - `src/research/ngx/__init__.py`: marks the Nigerian folder.
> - `src/research/ngx/pdf.py`: reads results PDFs and checks itself two ways.
> - `src/research/ngx/manual.py`: reads hand-entered figures, each with a source.
> - `scripts/read_ngx_inbox.py`: drafts figures from every waiting PDF.
> - `.github/workflows/ngx-inbox.yml`: runs it when a PDF is uploaded.
> - `ngx/companies.yaml`: Nigerian company details and typed-in prices (empty for now).
> - `ngx/inbox/.gitkeep` and `ngx/extracted/.gitkeep`: keep the two folders in place.
> - `ngx/README.md`: how to add a Nigerian company.
> - `ngx/email-to-ngx.md`: a draft email asking NGX about personal-use access.
> - `tests/test_ngx_manual.py`, `tests/test_ngx_pdf.py`: their tests, with made-up numbers.

## Step twelve: checks and the last documents

`.github/workflows/tests.yml` checks every change: the linter, then all the tests, then all the tests again with the internet blocked. So a test that secretly calls a real service fails straight away.

`ARCHITECTURE.md` explains how it's all put together, for whoever changes it next. And `DEPLOY.md` is the setup guide: make the repo private, add the secret keys, create the site on Cloudflare, put the email login in front, flip the safety switch. About half an hour, once.

> **📁 Files we just created**
> - `.github/workflows/tests.yml`: lint, tests, and tests with the internet blocked.
> - `ARCHITECTURE.md`: how it's built, for the next person.
> - `DEPLOY.md`: the half-hour setup guide.

## How's it doing?

This isn't a prediction model, so there's no accuracy score. The question is: are the numbers right?

436 tests pass, with one skipped, in about 12 seconds. They use real filings from five companies, with the sums written next to the expected numbers. The PDF reader was tried on one real report, NGX Group's own first-quarter 2026 results, and read every main figure correctly, with both checks passing.

## What's still missing?

- **No Nigerian company is entered yet.** The tools are ready; the companies aren't.
- **Nigerian prices are typed in by hand.** The paid options are written down, and choosing one is the owner's call.
- **The PDF reader has only met one real report.** Scanned PDFs can't be read at all.
- **Whether the site is live isn't shown in the repo.** The setup steps are written, but they're done by hand.
- **Some leftovers from the first life**, like the trading-day calendar, are tested but unused.

## Let's put it all together

So let's look at it in one breath.

We **checked the rules first**, and refused to scrape a site that forbids it, even after a bug nearly told us otherwise. We **kept a solid data layer** from the project's first life, and fed it **US prices** with three traps guarded, and **US accounts** in the periods people actually use.

We made **every number carry its evidence**, or say why it's missing. We worked out **29 figures**, **8 warning checks**, five years of P/E history and **peers**. We wrote **explanations by hand** and tested that they never give advice.

Then we built a **plain website**, a **nightly robot** that rebuilds it for free and alerts me, and a **safe way in for Nigerian companies** through their own PDFs, with a person always checking.

Notice how it links. Every part serves one goal: a number I can trust, and understand, without anyone telling me what to do with it.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
