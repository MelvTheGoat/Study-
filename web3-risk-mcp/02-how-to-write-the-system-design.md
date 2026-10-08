# web3-risk-mcp: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

---

## Step 1: Requirements (2 min)

One line:
> "A read-only tool server that lets any AI assistant check a crypto address for scam risk and explain the score."

**Functional**
1. Given an address and chain, detect if it's a wallet, token or contract.
2. Check tokens (honeypot, taxes, owner powers, liquidity), contracts (verified, proxy, who controls it), wallets (age, behaviour, labels) and fund flows (1–2 hops).
3. Return a 0–100 score, a level, a verdict, confidence, and a reason for every point.
4. Expose it through MCP: tools, a resource (the method doc) and a prompt.

**Non-functional**
- **Read-only:** never ask for keys, never sign or send.
- **Explainable:** no hidden weights.
- **Resilient:** one data source down doesn't break the result.
- **Free APIs:** respect rate limits.
- **Honest:** missing data lowers confidence, never lowers the score silently.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Chains | 6 EVM chains (including Arc) |
| Tools | 6, plus 1 resource and 1 prompt |
| Rules | 85 (11 decisive, 4 trust) |
| Decisive floor | 75 |
| Cache TTL | 5 minutes (default) |
| Trace sample | 100 txs root, 50 at hop 2, fan-out 3 |
| Eval set | 34 addresses (12 risky, 22 safe) |
| Tests | 102 |

**Say:** "Request volume is low. The hard parts are external API limits and explainability."

---

## Step 3: High-level boxes (2 min)

```
[AI assistant] --MCP--> [Server: tools/resource/prompt]
                              |
                     [score_risk orchestrator]
                  /       |          |        \
            [wallet]  [token]  [contract]  [trace]   --> Findings
                  \       |          |        /
               [HTTP layer: cache, rate limit, retry, redact]
                 |        |          |        |
            [Etherscan] [GoPlus] [DexScreener] [RPC (allow-list)]
                              |
                    [Pure scorer: rules -> score + reasons + confidence]
```

---

## Step 4: Deep dive (8–10 min)

### 4a. MCP layer
- Tools are marked `readOnlyHint`. Transports: stdio and streamable HTTP.
- The resource `risk://scoring-method` is generated from the rule table.
- The prompt `investigate_address` gives the assistant an investigation plan.

### 4b. Findings vs scoring (the key design)
- Analysis code **only describes**: `Finding(id, severity, reason, source)`.
- The scorer is a **pure function**: `score_findings(findings, sources) -> ScoreResult`.
- Why: easy to test, easy to explain, and weights change in one place.

### 4c. Scoring algorithm
1. Look up each finding ID in `RULES` (falling back to severity points).
2. Group duplicates (e.g. "not verified" from two sources) and keep the max.
3. Sum and clamp to 0–100.
4. If any decisive finding exists (honeypot, sanctioned, exploiter, phishing, fake token...), `score = max(score, 75)`.
5. Confidence: all sources OK → high, ≤50% failed → medium, else low.
6. Level bands: 75+ critical, 50+ high, 20+ medium, else low.

### 4d. Data layer
- One shared async HTTP client with a per-source rate limit, cache, and retries on timeout/429/5xx with backoff and jitter.
- Keys removed from logs, cache keys and errors.
- The RPC client only allows a list of read methods.

### 4e. Contract inspection
- Verified source? Proxy (read EIP-1967 storage slots)? Owner type (EOA, multisig, renounced)?
- Unverified: scan bytecode for known 4-byte selectors (mint, blacklist, pause, setFee...).

### 4f. Fund tracing
- Hop 1: all direct counterparties from recent history. Hop 2: the busiest 3.
- Don't expand exchanges, protocols or burn addresses.
- Check each node against the local list, and screen the closest with GoPlus.

### 4g. Evaluation
- 34 labelled addresses. Metrics: ROC AUC, and precision/recall at 50.
- Run twice: with and without the local bad list (to avoid "the list did all the work").
- A cassette records responses so anyone can replay offline.
- First live run (30 Sep 2026): ROC AUC 0.981, 1 false alarm, 1 miss. Without the list: 0.958, 3 misses.

---

## Step 5: Bottlenecks (2 min)

1. **Free API rate limits.** Handled by the cache, per-source limits and backoff.
2. **Source outages.** Handled by per-source wrapping, data gaps and confidence.
3. **History limits** (Etherscan free plan on Base/BNB). Reported as gaps.
4. **Trace fan-out.** Capped by the history limit, fan-out and hop count.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Rule table | ML model | Explainable, no labelled data at scale | Hand-set weights |
| Group max | Sum everything | No double counting across sources | Loses some corroboration signal |
| Decisive floor | Pure sum | Trust signals can't hide a honeypot | Less nuance |
| Sampled traces | Full graph | Free-tier limits and latency | Can miss old links |
| MCP | Custom API | Works in every MCP client | Tied to the MCP ecosystem |
| Read-only | Wallet actions | Safety | Can't "fix" anything, only warn |
