# Stocks: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Stocks

1. **A private research site for understanding one company at a time, US or Nigerian. It explains every number and never says buy or sell.**
   *Why it matters:* it's a learning tool, not a tip sheet.

2. **Every figure carries its source and date, or a sentence saying why it's missing. Nothing is guessed (no P/E for a loss).**
   *Why it matters:* every number can be checked.

3. **US accounts come from the SEC's company facts: latest filing wins per period, labels chosen per period, TTM = FY + YTD − prior YTD.**
   *Why it matters:* it handles restatements, label changes and missing periods.

4. **US prices come from Twelve Data's free plan: incremental, 700 of 800 requests a day, with guards against an inverted split field.**
   *Why it matters:* free APIs give wrong numbers silently if misread.

5. **29 figures and 8 warning checks, each with its rule written out.**
   *Why it matters:* it separates cheap-and-overlooked from cheap-and-failing.

6. **Context: five years of P/E at quarter ends, and peers by industry code (US peers found automatically from SEC data).**
   *Why it matters:* a number only means something against something.

7. **Explanations are hand-written and tested: every figure covered, short sentences, no advice words.**
   *Why it matters:* fixed text can be checked; generated text can't.

8. **Nigeria: NGX forbids automated collection, so results PDFs are uploaded, read, checked two ways and marked unchecked; prices are typed in.**
   *Why it matters:* it respects terms, and keeps a person in the loop.

9. **A nightly GitHub Actions build makes a static site on Cloudflare Pages behind a login, with ntfy phone alerts. All free.**
   *Why it matters:* no server, no cost, and private because of the price licence.

10. **436 tests pass, using real SEC filings. It started as an AI agent and was turned into this on 6 Oct 2026.**
    *Why it matters:* the numbers are verifiable, and the change of direction was deliberate.
