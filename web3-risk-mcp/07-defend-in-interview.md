# web3-risk-mcp: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

---

## 60-second pitch

> "People ask AI assistants 'is this token safe?', and without data the assistant guesses. I built web3-risk-mcp, an MCP server that gives any MCP-compatible assistant real tools to check a wallet, token or smart contract across five EVM chains. It pulls from Etherscan, GoPlus, DexScreener and public RPC nodes, checking for honeypots, owner powers, taxes, liquidity, proxies and fund links to mixers or exploiters.
>
> The key design choice: analysis only produces findings, and a separate pure scorer turns them into a 0–100 score using a public rule table, with a reason for every point. Duplicate findings from different sources count once, decisive findings like a honeypot set a floor of 75, and missing data lowers confidence instead of the score. It's read-only by design: the RPC client refuses any non-read method. There are 102 tests, and a 34-address evaluation that runs with and without my local bad-address list. The live evaluation numbers aren't in yet."

---

## Questions and honest answers

### 1. "Why MCP instead of a normal REST API?"
MCP is the standard way AI assistants call tools. Write it once and it works in any MCP client. It also runs over HTTP, so a normal service could use it too.

### 2. "Why rules instead of machine learning?"
Two reasons. There isn't a big labelled dataset, and the whole point is explainability: every point has a reason a beginner can read. The honest cost is that the weights are hand-set. With a larger labelled set, I'd fit weights (e.g. logistic regression over finding IDs), which keeps each contribution explainable.

### 3. "How does the score work exactly?"
Each finding ID maps to points in an 85-rule table. Findings about the same problem share a group, and only the max counts. Sum, then clamp to 0–100. If any of 11 decisive findings exist, the score is at least 75. Confidence is high if every source answered, medium if up to half failed, low otherwise. Levels: 75+ critical, 50+ high, 20+ medium.

### 4. "How do you know it works?"
The code is tested: 102 tests with all HTTP mocked. How well the score separates risky from safe addresses is **not measured yet**. The harness is ready, with 34 labelled addresses, ROC AUC, and precision and recall at 50. It runs twice, once without the local bad list, so the list can't carry the result. I haven't done the live run.

### 5. "Why the second eval run without the local list?"
Six of the 12 risky items are on my local list. With the list on, the score would catch them trivially. The second run shows what GoPlus, contract analysis and behaviour catch on their own. That's the more honest number.

### 6. "What if a data source is down?"
Each call is wrapped. You get partial results, `data_gaps` says what failed, and confidence drops. The score never quietly treats "no data" as "safe".

### 7. "Why is a low score not 'safe'?"
Because it means "no red flags in the data we could check". A brand-new scam that GoPlus hasn't scanned and that isn't on any list can score low. The verdict text says exactly that.

### 8. "How do you inspect unverified contracts?"
Solidity loads each function selector with PUSH4 (0x63), so I scan the bytecode for known 4-byte selectors like mint, blacklist, pause and set-fee. It's a heuristic. Renamed or custom functions can slip through. Proxies are detected with Etherscan's flag or the EIP-1967 storage slots.

### 9. "How does fund tracing avoid blowing up?"
It's a sample: the latest 100 transactions at the root, 50 at hop 2, the 3 busiest counterparties expanded, and no expansion through exchanges, protocols or burn addresses, because they touch everyone.

### 10. "What's the safety model?"
No keys, no signing, no sending. Tools carry `readOnlyHint`. The RPC client has an allow-list of read methods and raises on anything else. So even a confused assistant can only read public data.

### 11. "How would you scale it?"
Shared Redis cache, paid API plans or my own indexer for full history, background jobs for deep traces, auth and quotas on HTTP, per-source monitoring, and a much larger labelled set to fit weights.

### 12. "What would break first?"
Free API limits and Etherscan's free-plan gaps (no history on Base/BNB). Then GoPlus coverage for brand-new tokens.

### 13. "Why take the max per group rather than adding?"
To avoid double counting the same fact from two sources. The trade-off is losing corroboration: two sources agreeing doesn't add confidence to the score. I'd consider a small bonus for agreement.

### 14. "How do you keep the docs honest?"
The method document is generated from the rule table, and a test fails if the committed doc doesn't match the code.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| No live eval results | "What's your AUC?" | "Not measured yet. The harness and dataset are ready, with a replay cassette for reproducibility." |
| Hand-set weights | "Why 60 for honeypot?" | "It's a deal-breaker pattern, and it's also decisive (floor 75). The weights are public and will be checked by the eval." |
| Tiny eval set | "34 items proves nothing." | "Agreed, it's a sanity check. Growing it is the next step before fitting any weights." |
| Relies on GoPlus | "Isn't GoPlus doing the work?" | "For token flags, largely yes. My value is combining sources, contract and trace analysis, and explainable scoring. The no-list eval pass measures the rest." |
| Heuristic bytecode scan | "Easy to evade." | "Yes. It flags known patterns, and it's labelled as possibly incomplete in the output." |
