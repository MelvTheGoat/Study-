# web3-risk-mcp: Explained Simply

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A safety checker that AI chat assistants can use to tell you whether a crypto coin, crypto account or crypto program looks like a scam, before you put money into it.

## The everyday comparison

Think of buying a used car. Before you pay, a good mechanic:
- checks the car's history (was it stolen? in a crash?),
- looks under the bonnet for hidden problems,
- and gives you a report: "I'd avoid this one, because the brakes are worn and the mileage has been changed."

My tool is that mechanic for crypto. It checks:
- **the history of an account:** has money come from hackers or from services that hide where money came from?
- **the rules inside a coin:** can you actually sell it once you've bought it? Can the owner secretly create more coins or charge a 99% fee to sell?
- **who's in control:** can one person change the rules whenever they like?

Then it gives a score from 0 (no red flags found) to 100 (very likely dangerous), and it explains **every single point** of the score in plain English.

## Why AI assistants?

Many people now ask chat assistants "is this coin safe?" Without real information, the assistant can only guess. My tool plugs into these assistants using a standard called MCP (a common "plug" that lets AI assistants use outside tools), so the assistant can look at real evidence instead of guessing.

## It can't touch your money

The tool only **reads** public information. It never asks for passwords or secret keys, and it cannot send or move money. It's built so that it's impossible, not just "not allowed".

## Honest limits

- A low score means "no red flags found", not "guaranteed safe". A brand-new scam might not show up yet.
- I've prepared a test with 34 known good and bad addresses, but I haven't run the live test yet, so I don't have accuracy numbers to share.

## What this shows about me

- I understand how crypto scams actually work.
- I build tools that explain their answers instead of hiding behind a number.
- I design for safety first.
- I measure honestly, including testing whether my tool still works without its "cheat sheet" of known bad addresses.
