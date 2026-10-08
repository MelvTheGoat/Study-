# A research tool that explains every number: US and Nigerian companies, without the advice

Repo: https://github.com/MelvTheGoat/Stocks

## Why I built it

Most people judge a share by its chart. The chart is the least useful thing about a company. What matters is whether it makes money, turns that money into cash, can pay its debts, and what you're paying for all of that.

Those answers are in company reports, written in a language most people never learned. I wanted a tool that shows me the numbers that matter and explains each one while I look at it: what it is, what to compare it with, and how it usually misleads. The aim is that, after a while, I don't need the explanations.

There's one hard rule: **it never tells me to buy or sell.** Ask it "should I buy this?" and it lays out the inputs to that decision instead. Making the decision myself is how I learn to make it well.

It covers US companies and Nigerian companies listed on the Nigerian Exchange (NGX), because that's what I want to understand.

## Where it stands, honestly

| Part | State |
|---|---|
| US company pages (figures, checks, P/E history, peers, latest report) | built and tested |
| Home, compare and data-freshness pages | built |
| Nightly build, Cloudflare publishing behind a login, phone alerts | built; setup steps in `DEPLOY.md` |
| Nigerian companies from PDFs | built; no company entered yet |
| Nigerian prices | typed in by hand for now |
| Tests | 436 pass, 1 skipped |

This repo didn't start here. From 18 September it was an "eval-first" AI agent that would answer questions about stocks, with a question set, graders, a retrieval baseline and a GPU runner. On 6 October I removed the agent and the eval, kept the data layer, and turned it into this. A tool I'd use every day was worth more than a benchmark.

## The problem, broken down

To make a page I'd actually trust, I need:

1. **Numbers that carry their own evidence**: a source and a date, or a reason they're missing.
2. **Accounts in the periods people use**, like the last twelve months, which no filing hands over directly.
3. **Prices that are correct after splits**, from a free source that's easy to misread.
4. **Context**: today's figures against the company's own past and against similar companies.
5. **Explanations I can check**, which never slip into advice.
6. **Nigerian data I'm allowed to use.**
7. **All of it free, private, and automatic.**

## How it works

### Every number knows where it came from

The core type is a `Figure`. It's not just a value: it carries the unit, the currency, the date it describes, its sources, any notes on how it was worked out, and whether it's stale. If there's no value, it carries a `missing` sentence, and the page shows that sentence instead of a number. A test checks that every figure for every test company has one or the other.

This is the price-to-earnings ratio refusing to guess, straight from the code:

```python
if profit.value <= 0:
    return Figure.absent(
        "pe",
        "multiple",
        f"No P/E: the company made a loss over the twelve months to "
        f"{fmt.long_date(profit.end)}. A P/E divides by profit, so it needs a profit.",
```

The same rule runs everywhere: no price-to-book for negative equity, and no dividend yield of zero unless both the price data and the cash flow statement agree nothing was paid.

### US accounts, in the periods people want

US companies' accounts come from the SEC's "company facts" files: every number a company has tagged in its filings, one file per company, free and public. The same number appears many times, in the original report, as last year's column in the next one, and again if it's restated. So for each period the most recently filed value wins.

Companies also change the labels they use. Apple's revenue label changed in 2018. So each figure has a list of labels, best first, chosen separately for each period. Every label is there because a real filing needed it.

Filings don't give the periods readers want. A cash flow statement only reports the year so far, and nobody reports the fourth quarter on its own. So the tool works them out: the last twelve months is the last full year, plus this year so far, minus the same part of last year. The fourth quarter is the full year minus nine months.

### Prices, and three traps

US prices come from Twelve Data's free plan: 800 requests a day, 8 a minute. The nightly job only asks for days it doesn't have, and stops at 700 so there's room to spare. Dividends and splits are checked weekly.

That API has three traps, and each one gives confidently wrong numbers rather than an error. A split arrives with a `ratio` field that's the wrong way round: reading it would make Apple's split-adjusted returns wrong by a factor of sixteen. So the factor comes from `from_factor / to_factor`, and is checked against the number written in the description. The end date of a request is exclusive, and there's no adjusted close, so adjusting for splits is done in my own code.

### Eight warning checks

A share can look cheap because the market missed something, or because the market saw something. The checks look for the second kind. Each one shows its rule, so I can disagree with it.

| Check | Rule |
|---|---|
| Share price | Flags a fall of more than half over three years |
| Sales trend | Concern if sales shrank over 5% a year over three years |
| Profitability | Concern if it lost money this year and last |
| Cash burn | Concern if the cash lasts under 18 months |
| Debt payments | Concern if operating profit covers interest under 1.5 times |
| New shares | Concern if the share count grew over 20% in three years |
| Ease of selling | Whether enough shares trade to sell yours |
| Value against peers | P/E against similar companies, as context, with no threshold |

### Context, not verdicts

"Is 30 times earnings expensive?" has no answer on its own. So each page shows the P/E at every quarter end for five years, the middle half of that range, and where today's sits. It also compares with similar companies, picked by industry code. For US companies, the tool finds those peers itself: the SEC's list of companies in that industry, cut to those still on a major exchange, ranked by closeness in sales.

### Explanations written by hand

Every figure has three short pieces of hand-written text: what it is, what to compare it with, and how it misleads. I chose fixed text over generated text, because being wrong about what a figure means is worse than not showing it. Fixed text can be read carefully and tested.

The tests check every figure has an explanation, the sentences stay short, and no text ever contains advice words like "buy", "bargain", "undervalued" or "you should". The sentences about a particular company are built only from its own figures, so each one traces back to a number on the page.

### A site with nothing to run

The site is plain HTML, built every weekday night by GitHub Actions and published on Cloudflare Pages. Charts are drawn as pictures when the page is built, with the same numbers in a table underneath. Each page opens with a few plain sentences; the detail opens when you tap a heading.

The price data is licensed for personal use, so the site sits behind Cloudflare Access, which emails a one-time code. There's also a safety switch: the nightly job won't publish at all until a setting says the login is in place. Each night it also compares watched companies with the night before, and sends one phone alert through ntfy if something changed.

## The hard part: data I'm allowed to use

Every source has to have an entry in `DATA_SOURCES.md` before anything is collected. For the Nigerian Exchange, that entry says no. Its terms forbid "any systematic or automated data collection" without written consent, and forbid republishing its data. Its licensed data service costs money: end-of-day prices are N125,000 a year.

Reading those terms nearly went wrong. The site sits behind a firewall that blocked my development machine, so I checked from a GitHub runner instead. The first run reported no rules about automated access. That was false: the page came back compressed in a format the runner couldn't decode, and a keyword search over binary noise naturally found nothing.

I fixed the decoding, and the clauses were there in the first pass. Had I believed the first run, I'd have recorded permission that doesn't exist.

So for Nigerian companies, a person downloads the results PDF, the way you'd read it anyway, and uploads it. A workflow reads the main figures, notes the page each came from, and checks its own work two ways: profit before tax must match on the income and cash flow statements, and the balance sheet must balance. Everything is marked "not yet checked" until a person compares it with the PDF, and the PDF is deleted so nothing is republished. Prices are typed in by hand and turn amber after four days.

## What I learned

- **Make every number carry its evidence.** A source and a date turn "trust me" into "check me".
- **Saying "missing, because…" beats a guess.** A blank with a reason is honest.
- **Fixed text can be tested; generated text can't.** For explanations, that matters more than variety.
- **Read the terms properly before building.** And check that you actually read them.
- **Free APIs hide traps.** The worst bugs give believable wrong answers, not errors.
- **Change direction when the goal changes.** The agent was interesting; this is useful.

## What's next

1. Enter the first Nigerian companies from their results PDFs.
2. Decide on a licensed Nigerian price feed, or hear back from NGX about personal-use access.
3. Try the PDF reader on more reports; so far it's been tested on one real one.
4. Finish the setup in `DEPLOY.md` and use it every day.
