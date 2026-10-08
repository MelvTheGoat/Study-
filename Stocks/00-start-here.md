# Stock Research: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: this project changed direction. It started in September 2026 as an AI helper that would answer questions about stocks. On 6 October 2026 that was dropped, and it became a research website instead. This file describes the website.

## 1. The Problem

Most people judge a share by its price chart. But a chart can't tell you if a company is making money, running out of cash, or drowning in debt. The numbers that can tell you are spread across long reports, full of words most people never learned.

This project builds a private website for one person. You look up a company, American or Nigerian, and it shows you the important numbers. Next to each one, it explains what the number means, what to compare it with, and how it most often misleads.

## 2. The Big Idea

The big idea is "teach while you look". The site doesn't just show numbers; it explains each one while you read it. The goal is that you get better at reading companies by using it, until you don't need the explanations.

And it **never tells you to buy or sell**. If you want to know whether to buy, it lays out the inputs instead: what you pay for the company's profit, which way the business is going, whether it can pay its debts, and what changed recently. The decision stays yours, because making it yourself is how you learn.

## 3. What's Built and What's Planned

**Built today:**
- Company pages for US companies, with figures, explanations, warning checks, price-to-profit history and a comparison with similar companies.
- A home page with your watchlist, a page to compare up to four companies, and a page showing how fresh each company's data is.
- A nightly job that fetches new data, rebuilds the site, and sends you a phone alert if something changed.
- A way to add Nigerian companies: upload a results PDF and the tool drafts the figures for you to check.
- 436 automatic tests.

**Not done yet:**
- No Nigerian company has been entered yet.
- Nigerian share prices have to be typed in by hand. Paying for a licensed price feed is left as the owner's decision.
- The setup steps for putting the site online are written, but the repo doesn't show whether it's live yet.

## 4. How It Works, Step by Step

**Step 1: Choose the companies.** A simple list file, `watchlist.yaml`, names the companies to follow. You can edit it right on GitHub. It currently watches Apple, Coca-Cola and McDonald's, and tracks 18 other big US companies.

**Step 2: Fetch the data every weekday night.** A free GitHub robot wakes up after the US market closes. It downloads company accounts from the SEC, the US regulator, and new share prices from Twelve Data, a price service with a free plan.

**Step 3: Find similar companies.** For each watched US company, it finds others in the same industry, still trading on a major exchange, and closest in size by sales.

**Step 4: Work out the figures.** It turns the raw data into 29 figures, like the price-to-earnings ratio, profit margin and debt. Every figure carries where it came from and the date it describes. If a figure can't be worked out, it carries a sentence saying why.

**Step 5: Run the warning checks.** Eight checks look for signs that a cheap share is cheap for a bad reason, like shrinking sales or running out of cash.

**Step 6: Write the explanations.** Fixed, hand-written text explains each figure. Short sentences about this particular company are built from its own numbers.

**Step 7: Build the website.** It writes plain web pages, with small charts drawn in advance. Nothing needs to run on a server.

**Step 8: Publish it privately.** The pages go to Cloudflare Pages, behind a login that only lets the owner's email in.

**Step 9: Send alerts.** It compares tonight with last night. If a watched company filed a report, failed a check, moved sharply in price, or stopped getting data, it sends one message to your phone.

## 5. The Clever Parts

**Every number shows its source.** You can see which report each figure came from and the date it describes. Anything out of date turns amber: a price older than four days, or accounts older than 140 days.

**Nothing is guessed.** There's no price-to-earnings ratio for a company making a loss, because it would be meaningless. When a figure is missing, the page says why in plain words.

**Two layers on every page.** First come a few plain sentences, so you get the picture in thirty seconds. Then, under each heading, the detail opens when you tap it.

**Explanations are written by hand.** They are not made up fresh by an AI each time. A test checks that every figure has one, and that no explanation ever contains advice like "buy" or "undervalued".

**Respecting the rules.** The Nigerian Exchange's terms forbid automated collection. So the tool never scrapes it. Instead, you download a company's results PDF yourself and upload it, and the tool reads the figures out of it.

**It checks its own reading.** When it reads a PDF, it checks that profit before tax matches on two different statements, and that the balance sheet balances. Every figure is marked "not yet checked" until you've compared it with the PDF.

**A near miss, written down.** While checking the Nigerian Exchange's terms, a bug in reading the web page nearly made it look like there were no rules against collecting. The bug was found, and the rules were there. The project records this so it isn't repeated.

## 6. The Important Words

- **Share price**: the cost of one small slice of a company.
- **Market value (market cap)**: the price of the whole company: share price times the number of shares.
- **P/E ratio**: share price divided by profit per share. How many years of today's profit you're paying for.
- **Revenue**: all the money from sales, before costs.
- **Free cash flow**: cash left after running the business and paying for equipment.
- **Last twelve months**: the most recent full year of figures, worked out from the latest reports.
- **Peers**: similar companies to compare against.
- **Stale**: out of date.
- **SEC**: the US regulator that publishes every listed company's accounts.
- **NGX**: the Nigerian Exchange, where Nigerian companies' shares trade.

## 7. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **httpx**: downloads data from the SEC and Twelve Data.
- **Parquet and DuckDB**: store prices in files and read them with database queries.
- **Jinja2**: fills in the web page templates.
- **pypdf**: reads text out of results PDFs.
- **PyYAML**: reads the watchlist and the Nigerian company files.
- **pytest and ruff**: test the code and keep it tidy.
- **GitHub Actions**: runs the nightly build, the PDF reader and the tests.
- **Cloudflare Pages and Access**: host the site for free, behind a login.
- **ntfy**: sends free alerts to your phone.

## 8. How Good Is It?

This is a tool for reading companies, not a prediction model, so there's no accuracy score. What can be measured is whether its numbers are right.

436 automatic tests pass in about 12 seconds. They use real SEC filings from Apple, Coca-Cola, JPMorgan, GoPro and McDonald's. Wherever a test expects a number, a comment shows the sum from the filing, so a person can check it by hand. The PDF reader was tried on one real report, NGX Group's own first-quarter 2026 results, and read every main figure correctly.

## 9. What's Weak or Missing

- No Nigerian company has been added yet, and Nigerian prices must be typed in by hand.
- Scanned PDFs (pictures of pages) can't be read, so those figures must be typed in too.
- The PDF reader has only been tried on one real report.
- The US price data is licensed for personal use, so the site must stay private.
- The first nightly run can take up to an hour, because the free plan allows only 8 price requests a minute.
- Some older code from the AI-agent days is still there but unused, like a check that compares two price sources.

## 10. What This Project Shows You Can Do

- Turn messy official data into numbers people can trust, with sources.
- Design pages that explain, not just display.
- Respect data rules and licences, even when it's inconvenient.
- Build a free, automatic nightly system.
- Know when to change direction and cut what isn't working.

## 11. Ten Things to Remember

1. It's a private website for understanding one company at a time.
2. It covers US and Nigerian companies.
3. It explains every number, and never says buy or sell.
4. Every figure shows its source and date, or why it's missing.
5. Eight warning checks separate "cheap and overlooked" from "cheap and failing".
6. US accounts come from the SEC, and US prices from Twelve Data's free plan.
7. Nigerian figures come from PDFs you upload, checked two ways.
8. It rebuilds itself every weekday night, for free.
9. 436 tests pass, using real filings.
10. It started as an AI agent project and was changed on 6 October 2026.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
