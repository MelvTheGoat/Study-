# Crypto Risk Checker for AI Assistants: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

## 1. The Problem

Crypto scams are fast and final. If you send money to a scam token or wallet, there's no bank to call and no way to undo it.

More and more people now ask AI assistants, "Is this token safe?" But without real data, the assistant can only guess. A confident wrong guess could cost someone their savings.

## 2. The Big Idea

This project gives AI assistants a safe way to check crypto addresses using real data. It's a small "tool pack" that any compatible assistant can plug into.

You give it an address, and it returns a risk score from 0 to 100. Every single point comes with a plain reason and the source it came from. It can only *read* data. It can never move money.

## 3. A Few Crypto Basics First

A **blockchain** is a shared public record of every transaction. Nobody can secretly change it, and anyone can read it.

An **address** is like an account number on the blockchain. It can belong to a **wallet** (a person's account) or a **smart contract** (a program that runs on the blockchain by itself).

A **token** is a digital coin made by a smart contract. Anyone can create one in minutes, including scammers.

## 4. How It Works, Step by Step

**Step 1: The assistant asks.** An AI assistant calls the `score_risk` tool with an address and a blockchain name. It works with 5 blockchains, including Ethereum and Base.

**Step 2: Check the address is valid.** The tool checks the address is written in the right format.

**Step 3: Wallet or contract?** It asks the blockchain whether the address holds any code. No code means it's a wallet. Code means it's a contract or token.

**Step 4: Run the right checks, all at once.** For a token or contract, it checks for scam signs and hidden owner powers. For a wallet, it looks at its behaviour and follows its money one step. If one check fails, the others carry on.

**Step 5: Each check reports "findings".** A finding is one fact, like "the owner can create unlimited new tokens", with a reason and a source. The checks only describe what they see. They never give scores themselves.

**Step 6: The scorer adds up points.** Each finding matches a written rule worth some points. The scorer adds them up and keeps the total between 0 and 100.

**Step 7: The assistant gets the answer.** It receives the score, a level (low, medium, high or critical), every reason, and a list of any data it couldn't get.

## 5. The Scams It Looks For

- **Honeypot**: a token you can buy but can never sell.
- **Rug pull**: the creators suddenly take all the money out of a token's trading pool, leaving it worthless.
- **Hidden owner powers**: the owner can create new tokens, block certain holders, or freeze trading.
- **High sell tax**: a fee that quietly takes most of your money when you try to sell.
- **Few big holders**: one wallet owns most of the token and can crash its price.
- **Dirty money links**: money coming from mixers (services that hide where money came from), sanctioned addresses, or known thieves.

## 6. The Clever Parts

**Describing is kept separate from scoring.** The checks only report facts. A separate scorer turns facts into points. This makes both simple, testable and easy to explain.

**No double counting.** If two data sources both report "honeypot", it only counts once. Findings about the same problem share a group, and only the biggest one counts.

**Deal-breakers can't be outvoted.** 11 findings are "decisive", like a honeypot. Any one of them pushes the score to at least 75, which is "critical".

**Trusted things get points off.** Well-known, trusted tokens and long-established wallets get points taken off, so they don't look risky by mistake.

**Missing data never looks safe.** If a data source fails, the score isn't lowered. Instead, the "confidence" goes down. So a gap in the data can never make an address look safer.

**Safe by design.** Every tool is marked "read-only". The part that talks to the blockchain refuses any command outside a short list of "read" commands. So no mistake anywhere could ever send money.

**Keys stay secret.** API keys (secret codes for data services) are removed from logs, saved answers and error messages.

## 7. The Important Words

- **MCP (Model Context Protocol)**: a standard way for AI assistants to use outside tools, like a universal plug socket.
- **MCP server**: a program that offers tools to AI assistants. This project is one.
- **Finding**: one fact a check spotted, with a reason and source.
- **Severity**: how serious a finding is, from info to critical.
- **Decisive finding**: a deal-breaker that pushes the score to at least 75.
- **Confidence**: how much to trust the score, based on how many data sources answered.
- **Fund tracing**: following money from one address to the next.
- **Cache**: saved answers, reused for 5 minutes.
- **Rate limiting**: keeping requests under each free service's limits.
- **Precision and recall**: how many flagged addresses were truly risky, and how many risky ones were caught.

## 8. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **MCP Python SDK**: the official toolkit for building MCP servers.
- **httpx**: sends requests to outside data services.
- **Etherscan**: gives transaction history and contract code.
- **GoPlus**: gives scam flags for tokens and addresses.
- **DexScreener**: gives trading pool and money-in-pool data.
- **Public RPC**: lets it ask the blockchain questions directly, read-only.
- **pytest and respx**: run 102 automatic checks with pretend data services.
- **Docker**: packages it to run anywhere.
- **GitHub Actions**: checks every code change automatically.

## 9. How Good Is It?

102 automatic tests pass in about 5 seconds, with every outside service faked, so no keys are needed.

There's a test set of 34 hand-labelled addresses: 12 risky and 22 safe. But the first live run hasn't happened yet. So how accurate the scores are **hasn't been measured yet**.

## 10. What's Weak or Missing

- The points for each rule were set by hand, based on known scam patterns, not learned from data.
- 34 test addresses is a small set, so even when measured, the numbers will be rough.
- It only looks at recent history (the latest 100 transactions), and only the busiest paths.
- A brand-new scam that data services haven't seen yet can score low. A low score means "no red flags found", not "safe".
- Etherscan's free plan doesn't give account history on two of the blockchains.
- It only works with Ethereum-style blockchains.

## 11. What This Project Shows You Can Do

- Build tools that AI assistants can use safely.
- Combine several free data services reliably.
- Design a scoring system that anyone can check by hand.
- Build safety in at several levels, so mistakes can't cause harm.
- Be honest that "low risk" isn't the same as "safe".

## 12. Ten Things to Remember

1. It helps AI assistants check whether a crypto address is risky.
2. It's an MCP server, so any compatible assistant can use it.
3. It returns a score from 0 to 100, with a reason for every point.
4. Checks describe facts, and a separate scorer adds up points.
5. There are 85 written rules.
6. Nothing is counted twice.
7. Deal-breakers like honeypots push the score to at least 75.
8. Missing data lowers confidence, never the score.
9. It can only read data, never move money.
10. Its accuracy hasn't been measured yet.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
