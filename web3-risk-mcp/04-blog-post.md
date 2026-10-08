# Giving AI assistants a crypto fraud analyst: building web3-risk-mcp

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

## Why I built it

Crypto scams are common, fast and final. There's no bank to call and no undo button. The classic traps:

- **Honeypot tokens:** you can buy them, but the contract won't let you sell.
- **Rug pulls:** the creator pulls the trading money (the liquidity) and the price goes to zero.
- **Hidden owner powers:** the owner can mint new tokens, freeze your wallet, or raise the sell fee to 100%.
- **Dirty money:** a wallet that received funds from a hack or a mixer.

More and more people ask an AI assistant "is this token safe?" before buying. Without real data, the assistant can only guess, and a confident guess about money is dangerous.

I wanted to give assistants real evidence. So I built **web3-risk-mcp**, an MCP server. MCP (Model Context Protocol) is an open standard for connecting AI assistants to outside tools. Write a tool once, and every MCP client (desktop AI apps, Cursor and others) can use it.

## The problem

I had four goals:

1. **Real evidence** from several on-chain and security data sources.
2. **A score that explains itself:** every point has a reason and a source.
3. **Resilience:** free APIs are slow, rate-limited and sometimes down, but an investigation should still return something useful.
4. **Safety:** a tool handed to an AI must not be able to do harm.

## How it works

### Six tools, one resource, one prompt

| Tool | What it answers |
|---|---|
| `score_risk` | "How risky is this address?" Detects the type, runs the right checks, and returns 0–100 with every point explained |
| `get_wallet_profile` | Age, balance, activity, top counterparties, first funder, labels |
| `check_token_risk` | Honeypot signs, mint/blacklist/pause powers, taxes, holder concentration, liquidity and locks |
| `inspect_contract` | Verified? Upgradeable proxy? Who controls it? Risky functions in plain English |
| `trace_funds` | Money in and out for 1–2 hops, with links to mixers, sanctioned wallets, exploiters and phishing |
| `list_supported_chains` | Ethereum, Base, Arbitrum One, Polygon PoS, BNB Chain |

There's also a resource, `risk://scoring-method`, holding the full rule table, and a prompt, `investigate_address`, that gives the assistant a step-by-step plan and tells it how to explain the result to a beginner.

### Read-only by design

This is a safety feature, not a missing one. The server never asks for a private key or seed phrase, never signs or sends a transaction, and never holds funds. Every tool is marked `readOnlyHint`. And the RPC client has a second safety net: it refuses any JSON-RPC method that isn't on a short read-only allow-list.

```python
raise SourceError(NAME, f"Method {method} is not allowed. This server is read-only.")
```

The worst a confused assistant can do with these tools is read public data.

### Findings first, then scoring

This is the design decision I'm happiest with. The analysis code never produces a score. It only **describes** what it sees, as findings: a stable ID like `token.honeypot`, a severity, a plain-English reason, and the source.

A separate, pure function turns findings into a score:

1. Each finding ID has points in a public rule table (85 rules). Trust signals have negative points: for example, being a trusted token is −40.
2. Findings that describe **the same problem** share a group, and only the biggest one counts. "Source not verified" from GoPlus and from Etherscan counts once.
3. The total is clamped to 0–100.
4. **Decisive findings** (honeypot, sanctioned address, known exploiter, phishing, fake token and a few others, 11 in all) set a floor of 75, so no number of trust signals can hide them.

```python
floor_applied = decisive and score < DECISIVE_FLOOR
...
    score = DECISIVE_FLOOR
```

Every score also gets a **confidence**. If all sources answered, it's high. If up to half failed, medium. Otherwise low. **Missing data lowers confidence, never the score.** A report never quietly treats "no data" as "safe".

Because the scorer is pure and table-driven, it's easy to test, and anyone can recompute a score by hand. The method document is generated from the same table, and a test fails if the document drifts from the code.

### Resilient data fetching

All four sources (Etherscan V2, GoPlus, DexScreener and public RPC nodes) go through one shared HTTP layer:

- answers cached for 5 minutes by default
- a rate limit per source, under the free-plan limits
- timeouts, 429s and server errors retried with exponential backoff and jitter
- API keys removed from logs, cache keys and error messages

Each check is wrapped. If GoPlus is down, you still get Etherscan and DexScreener results, and the report lists exactly what failed in `data_gaps`.

## The hard parts

### Unverified contracts

Many scam contracts don't publish their source code. You can still learn a lot from the bytecode. Every public function has a 4-byte "selector" (a fingerprint of its name and inputs) stored in the bytecode. The contract inspector scans for known selectors like mint, blacklist, pause and set-fee, and turns them into plain English:

> This contract's source code is not published, so its behaviour is hidden... It is owned by a single wallet... it can create new tokens out of thin air, which dilutes every holder.

It also reads the standard EIP-1967 storage slots to spot upgradeable proxies, where the admin can swap in new code at any time.

### Tracing money without exploding

Following money can blow up fast: one wallet can touch thousands. So tracing is a **sample**, not a full audit. It takes the latest 100 transactions at the root and 50 at hop 2, expands only the 3 busiest counterparties at hop 2, and never expands exchanges, protocols or burn addresses, because they touch everyone and tell you nothing. Links to sanctioned wallets and exploiters are marked critical at hop 1 and lower at hop 2.

### An honest evaluation

It's easy to build a risk scorer that looks great on its own examples. So I built an evaluation set of 34 hand-checked addresses, each with a source for its label:

- **12 risky:** a known honeypot, the SQUID rug pull, 4 phishing wallets, 4 exploiter wallets (Ronin, Bybit, Euler, Wormhole) and 2 Tornado Cash pools.
- **22 safe:** major tokens on the first five chains, Uniswap and Aave contracts, a well-known personal wallet and an exchange hot wallet.

The script reports ROC AUC, precision, recall, false alarms and misses at a threshold of 50. And it runs **twice**: the second time with the local list of known bad addresses switched off. Six risky items are on that list, so without the second run the list could make the scorer look smarter than it is.

It also records every API response into a "cassette" file (with keys stripped), so anyone can replay the evaluation offline and get the same numbers.

**The results (first live run, 30 September 2026):** a ROC AUC of 0.981. At a score of 50, it caught 11 of the 12 risky addresses and flagged 1 of the 22 safe ones. With the local list switched off, the AUC was 0.958 and it missed 3.

The two mistakes are worth knowing. USDT scored 68, because its owner really can change balances. And the SQUID rug pull scored only 30, because the data services no longer flag it. That first run also led to some tuning: owner powers are now capped at 30 points together, because regulated stablecoins have all of them and aren't scams.

### Arc Safe Send

In October I added Arc, Circle's blockchain, where USDC is the native coin. On Arc, every check also reads Circle's USDC and EURC blocklists, and simulates a 1 USDC payment to see if it would go through. Nothing is sent. USDC moves are read from one system log, so a payment is never counted twice.

That became a live web page, https://arc-safe-send.onrender.com. You check an address, then pay from your own wallet. The server never sees a key. You can also save the check on Arc in a tiny contract, RiskAttestation, which is now deployed on Arc mainnet.

## What I learned

- **Separate description from judgement.** Findings plus a pure scorer made everything easier: tests, docs, explanations.
- **Explainability is a feature users feel.** "60 points: a test sale failed" is more useful than "risk: 0.87".
- **Floors beat sums for deal-breakers.** Some findings should never be outvoted.
- **Say what you couldn't check.** Confidence and data gaps stop "no data" from looking like "safe".
- **Safety by construction.** An allow-list in the RPC client means safety doesn't depend on every tool being written perfectly.

## What's next

- Add hand-checked Arc addresses to the evaluation set.
- Grow the labelled set, then consider fitting the weights while keeping each point explainable.
- Add chains, and a paid data plan or my own indexer for full history on Base and BNB Chain.
