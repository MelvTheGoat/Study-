# Stock Research: System Design for Beginners

This project is a private website that helps one person understand a company, American or Nigerian. It shows the key numbers from official reports, explains each one, and runs warning checks, but it never says buy or sell. It rebuilds itself every weekday night, for free. That makes it a great example of a simple "batch" system: do all the work once a night, then just show the results.

## Key Terms

- **Share price**: the cost of one small slice of a company.
- **Company accounts**: the official reports of a company's sales, profit, cash and debts.
- **Figure**: one number on the page, like profit or debt, with where it came from.
- **SEC**: the US regulator that publishes every listed US company's accounts for free.
- **NGX**: the Nigerian Exchange, where Nigerian shares are bought and sold.
- **Batch job**: a program that runs at a set time, does all its work, then stops.
- **Static website**: pages built in advance as plain files, with nothing running on a server.
- **Stale**: out of date.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The goal is understanding, not tips. The site must explain each number and never tell you to buy or sell.

**Step 2: Figure out the data.** US accounts are free from the SEC, and US prices are free from a service with limits. The Nigerian Exchange forbids automated collection, so Nigerian data must come another way.

**Step 3: Sketch the main parts.** Fetch the data, work out the figures, add explanations, build the pages, publish them privately.

**Step 4: Walk through one night.** Follow one company from "new report filed" to "updated page and a phone alert".

**Step 5: Decide how to know it works.** Check the figures against real reports by hand, and test that every number has a source.

**Step 6: Plan for problems.** Data can be late, free limits can run out, and licences can forbid sharing. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Show a page for each company, with **29** figures and **8** warning checks.
- Rebuild **every weekday night**, after the US market closes.
- Mark a price as stale after **4** days, and accounts after **140** days.
- Stay inside the free price plan: **800** requests a day, **8** a minute.

How long does a brand-new company's first price download take? Step by step:

1. One company's full price history costs one request.
2. The free plan allows 8 requests a minute.
3. So 21 companies need about 21 ÷ 8 ≈ **3 minutes** for prices alone. Dividends and splits add more requests, and the setup guide says the first run can take up to an hour.

### The Big Picture (Step 3)

```
   [Watchlist: which companies]
              |
              v
   [Fetch: SEC accounts + prices]  <---  [Nigerian PDFs, uploaded by you]
              |
              v
   [Figures + warning checks]
              |
              v
   [Explanations]
              |
              v
   [Build web pages]  --->  [Private website]
              |
              v
   [Compare with last night]  --->  [Phone alert]
```

Step by step:

1. Every weekday night, a free GitHub robot wakes up.
2. It reads the watchlist: the companies to follow.
3. It downloads new accounts from the SEC and new prices from Twelve Data, a price service.
4. It works out the figures and runs the warning checks.
5. It adds the explanations next to each figure.
6. It builds plain web pages and puts them online behind a login.
7. It compares tonight with last night, and sends a phone alert if something changed.

### The Main Parts (Step 3)

**Watchlist.** A short file listing the companies to follow. You can edit it on GitHub. If you make a typo, the build stops and names the mistake, so a company never silently disappears. It's like a shopping list that complains if you write something unreadable.

**Data Fetcher.** It asks the SEC for each company's accounts, and Twelve Data for prices. It only asks for prices it doesn't already have, so it stays inside the free limit. It's like only buying the milk you've run out of, not a whole new fridge.

**Figure Maker.** It turns raw reports into figures like profit margin and debt. Every figure remembers where it came from and the date it describes. If a figure can't be worked out, it says why, like "No P/E: the company made a loss". It's like a careful student who shows their working, and writes "can't answer, because…" instead of guessing.

**Warning Checks.** Eight checks look for trouble, like shrinking sales or running out of cash. Each one shows its rule, so you can disagree. It's like a car's dashboard lights, with the manual printed next to each one.

**Explanations.** Fixed, hand-written text explains each figure: what it is, what to compare it with, and how it misleads. A test makes sure none of it ever says "buy". It's like a museum label next to every painting.

**PDF Reader for Nigerian Companies.** You upload a company's results PDF. It reads the main numbers, checks they add up two ways, and marks them "not yet checked" until you've looked. It's like a helper who copies figures for you, but you still sign them off.

The PDF reader picks the right column by matching one number across two different statements (advanced - skip for now).

**Website.** Plain pages built in advance, with a few sentences at the top and details you tap to open. It sits behind a login, because the price data is for personal use only. It's like a private notebook that rewrites itself every night.

### How We Know It's Working (Step 5)

- **Tests**: 436 automatic checks pass in about 12 seconds.
- **Real reports**: tests use real SEC filings from Apple, Coca-Cola, JPMorgan, GoPro and McDonald's. Next to each expected number, a comment shows the sum, so a person can check it by hand.
- **Every number has a source**: a test checks every figure has a source or a reason it's missing.
- **No advice**: a test checks no explanation contains words like "buy" or "undervalued".
- **No internet in tests**: the tests run a second time with the internet blocked.

### What Can Go Wrong (Step 6)

- **The free price limit runs out.** The fetcher stops at 700 requests and carries on the next night.
- **Data goes out of date.** Old prices and accounts turn amber, and the summary says so.
- **The saved data is lost.** The next run downloads it again and catches up in a night or two.
- **Nigerian data isn't allowed to be collected.** So you upload PDFs yourself, and prices are typed in by hand.
- **A scanned PDF has no text.** The reader says so, and those figures must be typed in.

## Quick Recap

- It's a nightly batch job that builds a private website.
- Every figure shows its source and date, or why it's missing.
- Eight warning checks show their rules, and nothing ever says buy or sell.
- US data is fetched for free; Nigerian figures come from PDFs you upload.
- Tests use real reports, so the numbers can be checked by hand.
