# Stock Research: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow the path of the work: the money words, the data sources, working out the figures, the checks and context, the website, and running it every night.

---

## 1. The Money Words

### Share price and market value (market cap)
**What it means:** the share price is the cost of one slice of a company. Market value is the price of the whole company: share price times the number of shares.

**Why it's needed here:** a share price on its own says nothing about whether a company is cheap, because slices can be cut any size. So the site always shows the whole company's value too.

### Revenue, operating profit and net income
**What it means:** revenue is all the money from sales. Operating profit is what's left after running costs. Net income is the final profit after interest and tax.

**Why it's needed here:** they're the starting point for growth, margins and the P/E ratio.

### P/E ratio (price to earnings)
**What it means:** the company's value divided by its yearly profit. Roughly, how many years of today's profit you're paying for.

**Why it's needed here:** it's the main "how expensive is it?" number. The site works it out from the whole company's value and profit, and shows none when there's a loss, because dividing by a loss means nothing.

### Price-to-book
**What it means:** the company's value divided by what it owns minus what it owes (its "book value").

**Why it's needed here:** another view of price. It isn't shown when book value is negative.

### Dividend yield
**What it means:** the cash paid to shareholders each year, as a percentage of the share price.

**Why it's needed here:** it shows what you're paid to hold the share. A zero is only shown when both the price data and the company's cash flow statement agree nothing was paid.

### Free cash flow and cash conversion
**What it means:** free cash flow is the cash left after running the business and buying equipment. Cash conversion is how much of the profit turns into real cash.

**Why it's needed here:** profit can be an accounting number. Cash is what pays the bills, so the "cash burn" check uses it.

### Debt, net debt and interest cover
**What it means:** debt is borrowed money. Net debt is debt minus cash. Interest cover is operating profit divided by the interest bill.

**Why it's needed here:** the "debt payments" check asks whether profit covers the interest. Below 1.5 times is a concern.

### Cash runway
**What it means:** how long a company's cash would last if it keeps burning cash at today's rate.

**Why it's needed here:** under 18 months is a concern, because the company may need to borrow or sell new shares.

### Dilution
**What it means:** when a company issues new shares, each existing share becomes a smaller slice.

**Why it's needed here:** the "new shares" check flags a share count that grew more than 20% in three years.

### Stock split and bonus issue
**What it means:** a split cuts each share into several smaller ones. A bonus issue gives existing holders extra free shares, which has the same effect on price.

**Why it's needed here:** without adjusting for them, a price history looks like a crash. The project treats a Nigerian bonus issue as a split for this purpose.

---

## 2. The Data Sources

### SEC and EDGAR
**What it means:** the SEC is the US regulator. EDGAR is its free public system of company filings.

**Why it's needed here:** it's the source of every US company's accounts. It's free and public domain, but asks for a contact email with every request and allows 10 requests a second.

### 10-K and 10-Q
**What it means:** a 10-K is a US company's annual report; a 10-Q is its quarterly report.

**Why it's needed here:** only these are used, because other forms repeat the same numbers in less checked ways.

### Company facts
**What it means:** one SEC file per company, holding every number the company has tagged in its filings.

**Why it's needed here:** it's one download for all of a company's accounts. The same number appears many times in it, so the tool keeps one per period.

### XBRL tags (labels)
**What it means:** standard labels companies attach to each number in their filings, like "Revenues".

**Why it's needed here:** companies use different labels for the same thing, and change them over time. Apple's revenue label changed in 2018, so the tool keeps a list of labels for each figure.

### Restatement
**What it means:** a company correcting a number it reported before.

**Why it's needed here:** the latest filed value for each period wins, so trends use the corrected numbers.

### SIC code
**What it means:** a four-digit industry code the SEC gives each company, like 3571 for computer makers.

**Why it's needed here:** it's how peers are picked.

### Twelve Data
**What it means:** a share price service with a free plan.

**Why it's needed here:** it gives US prices, dividends and splits. The free plan allows 800 requests a day and 8 a minute.

### Terms of use and robots.txt
**What it means:** terms of use are a website's legal rules. robots.txt is a file saying which pages automated programs may read.

**Why it's needed here:** the Nigerian Exchange's terms forbid automated collection without written consent, so the tool doesn't collect from it. A permissive robots.txt isn't the same as permission.

### Personal-use licence
**What it means:** data you may use for yourself, but not share or publish.

**Why it's needed here:** the US price data is licensed this way, so the site must stay private.

---

## 3. Working Out the Figures

### Figure (with source and date)
**What it means:** in this project, a number plus where it came from, the date it describes, and whether it's out of date. Or, if there's no number, a sentence saying why.

**Why it's needed here:** it's the core rule: every number can be checked, and nothing is guessed.

### Last twelve months (TTM)
**What it means:** the most recent full year of figures. Worked out as last full year, plus this year so far, minus the same part of last year.

**Why it's needed here:** P/E should use a full year of profit that's as recent as possible. No filing reports this directly.

### Year to date
**What it means:** the figures from the start of the company's year until now.

**Why it's needed here:** quarterly cash flow statements only report this, so single quarters are worked out from differences.

### Stale data
**What it means:** data too old to trust without a warning.

**Why it's needed here:** a price older than 4 days, or accounts older than 140 days, turns amber on the page.

---

## 4. Checks and Context

### Warning check
**What it means:** a fixed rule that looks for a sign of trouble, with a status: ok, watch, concern, info, or unknown.

**Why it's needed here:** eight checks help tell a cheap-and-overlooked share from a cheap-and-failing one. A check that can't run says "unknown", never "ok".

### Value trap
**What it means:** a share that looks cheap because the business is in trouble.

**Why it's needed here:** it's exactly what the checks are designed to spot.

### Peers
**What it means:** similar companies, in the same industry.

**Why it's needed here:** figures mean more next to similar companies. Peers are picked by industry code, widening from four digits to three to two if needed.

### Median and middle half (interquartile range)
**What it means:** the median is the middle value. The middle half is the range holding the central 50% of values.

**Why it's needed here:** the P/E history shows its middle half over five years, and peers are compared against their median, which ignores extreme values.

---

## 5. The Website

### Static website
**What it means:** pages built in advance as plain files, with nothing running on a server.

**Why it's needed here:** it's free to host and has nothing to keep running. The pages are simply rebuilt each night.

### Template (Jinja)
**What it means:** a page layout with gaps that get filled in with data.

**Why it's needed here:** every company page uses the same layout, filled in with its own figures.

### SVG chart
**What it means:** a chart drawn as shapes in the page itself, not as an image file.

**Why it's needed here:** charts are drawn when the site is built, so nothing is calculated in the browser. The same numbers sit in a table underneath.

### Two tiers (progressive disclosure)
**What it means:** showing the short version first, and the detail only when asked.

**Why it's needed here:** plain sentences show first, so you get the picture in thirty seconds. Details open when you tap a heading.

### Access login (one-time PIN)
**What it means:** a login where the site emails you a code instead of using a password.

**Why it's needed here:** Cloudflare Access keeps the site private to the owner's email, as the price licence requires.

---

## 6. Running It Every Night

### Batch job and cron
**What it means:** a batch job runs at a set time and stops. Cron is the standard way to write the timetable.

**Why it's needed here:** the build runs at 01:17 UTC, Tuesday to Saturday, after each US trading day.

### Actions cache
**What it means:** a place GitHub keeps files between runs of a workflow.

**Why it's needed here:** it holds the downloaded data, so each night only fetches what's new.

### Rate limit and budget
**What it means:** the most requests a service allows in a time. A budget is a lower limit you set yourself.

**Why it's needed here:** the price fetcher stops at 700 of the 800 daily requests and carries on the next night.

### Idempotent write (merge on key)
**What it means:** a write that gives the same result however many times it runs.

**Why it's needed here:** re-running a fetch never creates duplicate prices.

### Parquet and DuckDB
**What it means:** Parquet is a compact file format for tables. DuckDB is a database that runs inside the program and can query those files.

**Why it's needed here:** prices are stored as Parquet files and read with SQL, with no database server.

### Fixture
**What it means:** saved sample data used by tests.

**Why it's needed here:** tests use real SEC filings for five companies, trimmed to the labels the tool reads, so they run without the internet.

### Push notification (ntfy)
**What it means:** a message sent to your phone. ntfy is a free service for this.

**Why it's needed here:** you get one message a night, only when something changed for a company you watch.
