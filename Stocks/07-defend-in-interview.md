# Stocks: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Stocks

---

## 60-second pitch

> "I built a private research site that helps me understand a company, US or Nigerian, before investing. It shows the figures that matter, like P/E, margins, cash flow and debt, and explains each one next to the number: what it is, what to compare it with, and how it misleads. It never gives buy or sell advice.
>
> Every figure carries its source and date, or a sentence saying why it's missing, because nothing is guessed. US accounts come from the SEC's company facts, with the last twelve months worked out from the filings. Prices come from Twelve Data's free plan. Eight warning checks, each with its rule shown, help tell cheap-and-overlooked from cheap-and-failing.
>
> It's a nightly GitHub Actions job that builds a static site on Cloudflare Pages behind a login, for free. For Nigerian companies, the exchange forbids automated collection, so I upload results PDFs and the tool reads them and checks itself two ways. There are 436 tests, using real SEC filings."

---

## Likely questions and honest answers

### 1. "What does it actually do today?"
For US companies, everything: a page per company with 29 figures, 8 checks, five years of P/E history, a peer comparison and what changed in the latest report. Plus a watchlist home page, a compare page and a data-freshness page, rebuilt every weekday night with phone alerts. For Nigerian companies, the PDF reader and the hand-entry format are built, but I haven't entered a company yet.

### 2. "Why no buy or sell advice?"
Because the point is to learn to judge companies myself. If it said "buy", I'd stop thinking. So it lays out the inputs: price against profit, against its own past and against peers, which way the business is going, whether it can pay its debts. A test even bans advice words like "undervalued" from every explanation.

### 3. "How do you get the last twelve months from filings?"
Last full year, plus this year so far, minus the same part of last year. The fourth quarter is the full year minus nine months, because nobody reports it alone. Cash flow quarters are differences of year-to-date figures. Anything worked out rather than read carries a note saying how.

### 4. "What about restatements and label changes?"
For each period, the most recently filed value wins, which picks up restatements. And each figure maps to a list of SEC labels, chosen per period, because companies change labels: Apple's revenue label changed in 2018. Every label in that list is there because a real filing needed it.

### 5. "Why is P/E computed from totals?"
Market cap divided by twelve months of net income. It gives the same answer as per-share figures, but doesn't depend on how each filing restated its per-share numbers after a split. And there's no P/E for a loss: the page says why instead.

### 6. "How do you handle the free price API?"
Twelve Data's free plan is 800 requests a day and 8 a minute. The sync only fetches days it doesn't have, stops at 700, and carries on the next night. It also guards three traps: the split `ratio` field is inverted, the end date is exclusive, and there's no adjusted close, so I adjust for splits myself.

### 7. "Why hand-written explanations instead of an LLM?"
Being wrong about what a figure means is worse than not showing it. Fixed text can be read carefully once and tested forever: every figure has one, sentences are short, and no advice words appear. The sentences about a specific company are built only from its own figures, so each one traces back to a number on the page.

### 8. "How do you find peers?"
By industry (SIC) code, widening from four digits to three to two if there aren't enough, and the page says which level it used. For US companies, peer search takes the SEC's list for that code, keeps companies on a major exchange, and ranks by closeness in revenue. Of the 100 companies filed under 3571, computer makers, only six still trade, so that filter matters. You can also pick peers by hand.

### 9. "Why a static site?"
It's a nightly batch, so there's nothing to compute per request. Static files are free to host, fast, and can't go down in interesting ways. Charts are SVG drawn at build time, with a table underneath, and every page works without JavaScript.

### 10. "Why not scrape the NGX website?"
Its terms forbid "any systematic or automated data collection" without written consent, and forbid republishing. Its licensed feed costs money. So a person uploads a results PDF; the tool reads it, checks profit before tax matches across statements and the balance sheet balances, marks everything unchecked, and deletes the PDF. Prices are typed in. I also drafted an email to NGX asking about personal-use access.

### 11. "Tell me about the near miss."
Checking NGX's terms from a GitHub runner first reported no rules about automated access. That was wrong. The page was compressed with brotli, the runner couldn't decode it, and a keyword search over binary noise found nothing. I fixed it to request only encodings that always decode and to check the body looks like text. The clauses were there. It's written up in `DATA_SOURCES.md`.

### 12. "How do you know the numbers are right?"
436 tests, using real SEC filings for Apple, Coca-Cola, JPMorgan, GoPro and McDonald's, with made-up prices so the arithmetic is easy. Where a test expects a number, a comment shows the sum from the filing. CI also runs everything with the network off.

### 13. "This started as an AI agent. Why change?"
The agent was an interesting experiment, but the question I actually care about is understanding companies I might invest in. A daily-use tool beat a benchmark. I kept the data layer, the Parquet and DuckDB store, the split arithmetic and the Twelve Data client, and removed the agent, eval and GPU runner.

### 14. "Why must it be private?"
The US price data is licensed for personal use. So it's behind Cloudflare Access with an email one-time code, and the nightly job has a safety switch: it won't publish until a setting confirms the login is in place.

---

## Weak spots and how to answer them

| Weak spot | Likely poke | Honest answer |
|---|---|---|
| No Nigerian companies yet | "So it's US only?" | "Today, yes. The PDF reader and entry format are built; entering companies is next." |
| Hand-typed NGX prices | "That doesn't scale." | "It doesn't. The paid options are costed in DATA_SOURCES.md. It's a licensing decision, not a code one." |
| PDF reader tested once | "One PDF isn't proof." | "Agreed. That's why every figure is marked unchecked until a person compares it, and it has two self-checks." |
| Is it live? | "Can I see it?" | "It's private by design. The setup checklist is in DEPLOY.md; I can show it built locally." |
| Leftover code | "Why is calendar.py still there?" | "It's from the agent version, tested but unused. It should be removed or put to use." |

**Rule:** there's no accuracy score, because it's not a prediction model. If asked "how accurate", talk about how the figures are verified: real filings, arithmetic in comments, and sources on every number.
