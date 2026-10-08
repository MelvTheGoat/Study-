# web3-risk-mcp: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

---

## Summary

A read-only MCP server (Python 3.11+, `mcp` SDK 2.x, async httpx) exposing 6 tools, 1 resource and 1 prompt for EVM address risk: wallet profiling, token risk, contract inspection (including bytecode selector scans and EIP-1967 proxy detection) and 1–2 hop fund tracing. Analysis emits findings. A pure, table-driven scorer (85 rules) turns them into a 0–100 score with per-point reasons, group de-duplication, a decisive floor of 75 and a source-based confidence. **170 tests pass (~10 s), with all HTTP mocked.** First live evaluation (30 Sep 2026) on 34 labelled addresses: ROC AUC 0.981, 1 false alarm, 1 miss. Built on 25 Sep 2026; Arc support, the Arc Safe Send web app and the RiskAttestation contract added 3–8 Oct 2026.

## Architecture

```
server.py            MCP registration: tools, resource, prompt
__main__.py          --transport stdio|http, --port
config.py            pydantic-settings (.env)
chains.py            6 chains (incl. Arc, 5042), address validation
web/                 Arc Safe Send: FastAPI app, limits, payment advice, static page
attestation.py       RiskAttestation call data and findings hash
contracts/           RiskAttestation.sol (Arc Foundry), checked deploy script
services.py          builds clients
clients/http.py      cache, per-source rate limit, retries (backoff+jitter), key redaction
clients/etherscan.py V2 multi-chain API
clients/goplus.py    token + address security
clients/dexscreener.py pools, liquidity
clients/rpc.py       eth_* read methods only (allow-list)
analysis/common.py   Collector: wraps calls, records SourceStatus
analysis/address.py  GoPlus address flags -> findings
analysis/wallet.py   profile -> findings
analysis/token.py    token checks -> findings
analysis/contract.py verification, proxy, ownership, selectors -> findings + summary
analysis/selectors.py known 4-byte selectors
analysis/trace.py    hop-1/hop-2 flow graph, risky links
analysis/score.py    score_risk orchestration (asyncio.gather)
scoring.py           RULES, score_findings, levels, confidence
method.py            renders the method doc from RULES
labels.py + data/known_addresses.json
evaluation.py        metrics + record/replay cassette
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| Findings ≠ score | `Finding(id, severity, reason, source)` → `score_findings()` | Pure, testable, explainable |
| Rule table | 85 rules. Unknown IDs fall back to severity points (critical 40, high 15, medium 8, low 2, info 0). | One place to tune |
| Grouping | Rules share a `group`, and only the max counts | No double counting across sources |
| Trust signals | 4 negative rules (e.g. `token.trusted` −40, `wallet.established` −10) | Well-known assets shouldn't look risky |
| Decisive floor | 11 decisive IDs → score ≥ 75 | Deal-breakers can't be outvoted |
| Confidence | 0 failed → high, ≤50% failed → medium, else low | Missing data never reads as safe |
| Levels | ≥75 critical, ≥50 high, ≥20 medium, else low | Simple bands with verdict text |
| Per-call isolation | `Collector.run(source, coro)` records ok/failed | Partial results with `data_gaps` |
| Read-only | `readOnlyHint` on tools + RPC method allow-list | Safety by construction |
| Doc from code | `render_method_doc.py`, with a drift test | Docs never go stale |
| Record/replay eval | Gzipped cassette, keys stripped | Offline, reproducible numbers |

## Algorithms

**`score_risk` orchestration:** first read the address's code over RPC. If there is code: run the token check, contract inspection and address labels in parallel (`asyncio.gather`), then call it a token if token data came back (pools, open-source flag or holders), otherwise a contract. If there's no code (`"0x"`, a wallet): run the wallet profile and a 1-hop trace in parallel. Merge findings, source statuses and data gaps, then score.

**Token checks:** GoPlus token security (honeypot simulation, buy/sell tax, owner powers such as mint, blacklist, pause, balance change, tax modifiable; open source; holder and LP stats) plus DexScreener liquidity.

**Contract inspection:** Etherscan source/ABI if verified. Otherwise scan bytecode for `PUSH4` selectors matching known risky functions. Proxy via Etherscan's flag or the EIP-1967 implementation/admin slots over RPC. Owner type: EOA, contract (possibly a multisig), or renounced.

**Trace:** build in/out flows from the latest 100 normal and token transfers. Hop 2 expands the top 3 counterparties (50 txs each). Skip exchange/protocol/burn categories. Label nodes from the local list and GoPlus-screen the closest (up to 6). Severity by category and hop (e.g. sanctioned: critical at hop 1, high at hop 2).

## How it's tested and evaluated

- **170 tests pass** (I ran them): chains, clients, config, contract, evaluation metrics, HTTP layer, `score_risk`, scoring rules, server registration, token, trace, wallet, Arc (replaying 70 recorded Arc mainnet responses), attestation, web API, CLI. Plus 8 Foundry tests on the contract. HTTP is mocked with respx, so no keys or network are needed.
- **CI:** ruff lint + format check, pytest on 3.11/3.12/3.13, Docker build.
- **Eval harness:** 34 items (12 risky, 22 safe) with label sources. Metrics: ROC AUC, and precision, recall, accuracy, TP/FP/TN/FN at threshold 50. Two passes (with and without the local list).
- **Eval results (30 Sep 2026, `eval/results.md`):** ROC AUC 0.981, precision 0.917, recall 0.917: 1 false alarm in 22 safe addresses (USDT, 68) and 1 miss in 12 risky (the SQUID rug pull, 30). Without the local bad list: AUC 0.958, 3 of 12 missed. Mean score risky/safe 82.2/9.5. Recorded responses are committed in `eval/fixtures/` for offline replay.
- **Rule table v3 (92 rules):** owner powers (mint, blacklist, pause, upgrade, withdraw, limits, single owner) capped at 30 together; `token.owner_can_change_balance` no longer decisive; Arc blocklist findings added as decisive (12 decisive in total); `address.official_contract` −30.

## Known weaknesses

- **Hand-set weights.** They follow scam patterns but aren't fitted.
- **Tiny eval set.** 34 items gives wide confidence intervals.
- **Heavy reliance on GoPlus** for token flags. If GoPlus hasn't scanned a new token, the score can be low.
- **Sampled traces:** recent history only, busiest paths only.
- **Selector scanning:** renamed or custom functions evade it, and false positives are possible from data that looks like `PUSH4`.
- **Etherscan free plan:** no history on Base/BNB.
- **Group-max loses corroboration.** Two independent sources agreeing adds no extra weight.
- **EVM only**, and no auth or quotas on the HTTP transport.
