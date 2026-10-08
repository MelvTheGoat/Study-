# Crypto Risk Checker for AI Assistants: System Design for Beginners

This project gives AI assistants a safe way to answer "Is this crypto token or wallet risky?" It's a small tool pack that any compatible assistant can use to check an address. It returns a risk score from 0 to 100, with a plain reason for every point. Crypto scams are fast and final, with no bank to call, so a clear, honest check really matters.

## Key Terms

- **Blockchain**: a shared public record of every transaction, which nobody can secretly change.
- **Address**: an ID on the blockchain, like an account number. It can belong to a **wallet** (a person's account) or a **smart contract** (a program).
- **Token**: a digital coin made by a smart contract. Anyone can create one, including scammers.
- **Honeypot**: a scam token you can buy but never sell.
- **MCP (Model Context Protocol)**: a standard way for AI assistants to use outside tools. Like a universal plug socket for AI.
- **API**: a way for programs to ask another service for data.
- **Finding**: one fact the checker spotted, like "the owner can create unlimited new tokens".

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The assistant needs real data, not guesses. And the user needs to see *why* something scored high.

**Step 2: Figure out the data.** Free services already know a lot: transaction history, token security flags, and trading pools. We combine them.

**Step 3: Sketch the main parts.** The assistant calls a tool, checks run in parallel, each check reports findings, and a scorer turns findings into a number.

**Step 4: Walk through one check.** Follow one address from "the assistant asks" to "score with reasons".

**Step 5: Decide how to know it works.** Test the score on addresses already known to be risky or safe.

**Step 6: Plan for problems.** Free services are slow and limited, some checks fail, and a low score could be mistaken for "safe". Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Offer **6** tools to any MCP-compatible assistant.
- Support **6** blockchains, including Ethereum, Base and Arc.
- Score from **0 to 100**, using **85** written rules.
- **Only read** data. It must never be able to send money.

How much history does a money trace look at? Step by step:

1. It reads the latest **100** transactions of the address.
2. It then follows the **3** busiest partners, reading **50** transactions each: 3 × 50 = **150**.
3. 100 + 150 = up to **250** transactions per trace.

### The Big Picture (Step 3)

```
   [AI assistant]
         |
         v
   [MCP server: 6 tools]
         |
         v
   [Checks: wallet, token, contract, trace]
         |                        |
         v                        v
   [Findings]            [HTTP layer: cache + limits]
         |                        |
         v                        v
   [Scorer: 85 rules]    [Free data sources]
         |
         v
   [Answer: score + reasons]
```

Step by step:

1. The assistant calls the `score_risk` tool with an address and a blockchain.
2. The server checks the address is in a valid format.
3. It looks up whether the address has code. No code means a wallet. Code means a contract or token.
4. The right checks run at the same time. If one fails, the others carry on.
5. Each check fetches data through the HTTP layer and reports findings, each with a reason and a source.
6. The Scorer adds up points for the findings and clamps the total between 0 and 100.
7. The assistant gets the score, a level (low to critical), every reason, and any data that was missing.

### The Main Parts (Step 3)

**MCP Server.** It's the front desk. Write the tools once, and any compatible assistant can use them. Every tool is marked "read-only".

**The Checks.** There are four. The wallet check looks at age and behaviour, and the token check looks for honeypots, high taxes and hidden owner powers. The contract check looks at who controls it, and the trace follows the money for one or two steps. They're like a mechanic's inspection list: brakes, tyres, engine, history.

**Findings.** The checks only *describe* what they see. They never give scores. It's like a doctor's notes ("high blood pressure") being kept separate from the final diagnosis.

**Scorer.** Each finding matches a rule worth some points. If several findings describe the same problem, only the biggest one counts, so nothing is counted twice. Some findings are deal-breakers, like a honeypot, and push the score to at least 75. Trusted, well-known tokens get points taken off, like a referee's scorecard anyone can recheck.

**HTTP Layer.** Every request to an outside service goes through here. It saves answers for 5 minutes, keeps under each service's free limits, retries failures, and removes secret keys from logs. It's like a careful assistant who makes all your phone calls for you.

**Read-Only Safety Net.** The part that talks directly to the blockchain refuses any command outside a short "read" list. So no mistake anywhere can ever send a transaction.

Scanning a contract's raw code for known function fingerprints works when its code isn't published (advanced - skip for now).

### How We Know It's Working (Step 5)

- **Tests**: 170 automatic checks pass in about 10 seconds, with every outside service faked, so no keys are needed.
- **Precision and recall**: of the addresses flagged risky, how many really were, and of the risky ones, how many were caught. On 34 hand-labelled addresses, it caught **11 of 12** risky ones, with **1 false alarm** in 22 safe ones.
- **Confidence**: how many data sources answered. If none failed, confidence is high. If more than half failed, it's low.

### What Can Go Wrong (Step 6)

- **Free services are slow or limited.** The cache and rate limits keep requests under each free plan's limits.
- **A data source fails.** Other checks continue, and confidence drops, but the score is never quietly lowered.
- **A low score is read as "safe".** A brand-new scam might not be known yet. A low score means "no red flags found", not "safe".
- **Secret keys leak.** Keys are removed from logs, saved answers and error messages.

## Quick Recap

- MCP lets any compatible AI assistant use the same tools.
- Checks describe what they see. A separate scorer turns that into a number.
- Every point comes with a reason and a source.
- Deal-breakers push the score to 75 or more, and nothing is counted twice.
- It can only read data. It can never move money.
