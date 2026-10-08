# web3-risk-mcp: What This Proves I Know

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

---

## 1. Model Context Protocol (MCP) and AI tool use

**Simple explanation:** a standard way for AI assistants to discover and call outside tools, read resources and use prompt templates.

**In this project:** 6 tools, 1 resource, 1 prompt, stdio and streamable HTTP transports, and `readOnlyHint` annotations.

**Also be ready to explain:** tools vs resources vs prompts, how a model chooses a tool (from its description and schema), transports, why tool descriptions matter, and tool safety (least privilege).

---

## 2. Blockchain fundamentals (EVM)

**Simple explanation:** accounts are wallets (keys) or contracts (code). Tokens are contracts. Everything is public.

**In this project:** detecting an account's type from its code, reading storage slots, ERC-20 tokens, DEX pools and liquidity.

**Also be ready to explain:** EOA vs contract, gas, ERC-20 approve/transferFrom, the AMM/DEX pool model, LP tokens, proxies (EIP-1967), multisigs, and function selectors (first 4 bytes of keccak256 of the signature).

---

## 3. Crypto fraud patterns

**Simple explanation:** the common ways people lose money.

**In this project:** honeypots, rug pulls (unlocked liquidity), mint/blacklist/pause powers, sell-tax traps, fake tokens, phishing wallets, mixers and exploiters.

**Also be ready to explain:** approval phishing, address poisoning, sanctions (e.g. OFAC and Tornado Cash), and why concentration of holdings matters.

---

## 4. Explainable, rule-based risk scoring

**Simple explanation:** add up points from clear rules so each part of the score can be explained.

**In this project:** 92 rules, a cap on owner powers, groups, decisive floor, trust signals, confidence, levels.

**Also be ready to explain:** scorecards in credit risk, rules vs ML trade-offs, calibration, how to fit rule weights (logistic regression) while keeping explainability, and thresholds and alert fatigue.

---

## 5. Evaluation of classifiers

**Simple explanation:** measure how well scores separate risky from safe.

**In this project:** ROC AUC, and precision, recall, accuracy and the confusion matrix at threshold 50. An ablation without the local list. Record/replay for reproducibility.

**Also be ready to explain:** ROC vs precision-recall curves (class imbalance), choosing thresholds, confidence intervals on small sets, label leakage (the local-list issue), and the base-rate fallacy.

---

## 6. Resilient API integration

**Simple explanation:** external APIs fail and limit you, so design for it.

**In this project:** cache with TTL, per-source rate limiting, retries with exponential backoff and jitter on timeout/429/5xx, secret redaction, and per-call isolation with data gaps.

**Also be ready to explain:** token-bucket rate limiting, idempotency, circuit breakers, and cache invalidation.

---

## 7. Async Python

**Simple explanation:** run many network calls at once without threads.

**In this project:** `asyncio.gather` across checks and sources, async httpx, pytest-asyncio.

**Also be ready to explain:** event loop basics, `gather` vs `TaskGroup`, cancellation and timeouts, and avoiding blocking calls in async code.

---

## 8. Graph traversal for fund tracing

**Simple explanation:** follow edges (transfers) out from a node (address), with limits.

**In this project:** 1–2 hop BFS-like expansion, fan-out cap, skip lists, severity by distance.

**Also be ready to explain:** BFS vs DFS, graph explosion, heuristics for "busiest path", and taint analysis methods (poison vs haircut).

---

## 9. Secure-by-design engineering

**Simple explanation:** make harmful actions impossible, not just discouraged.

**In this project:** no keys, an RPC allow-list, a non-root Docker image, and keys redacted from logs.

**Also be ready to explain:** least privilege, defence in depth, secret management, and prompt-injection risk when tools return untrusted text.

---

## 10. Packaging, CI and Docker

**In this project:** uv with a lock file, ruff, a pytest matrix on 3.11–3.13, Docker build in CI, and docs generated from code with a drift test.

**Also be ready to explain:** reproducible builds, lock files, and CI matrices.
