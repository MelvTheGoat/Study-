# web3-risk-mcp: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/web3-risk-mcp

1. **An MCP server that lets any AI assistant check a crypto wallet, token or contract for scam risk, across 6 EVM chains, including Circle's Arc.**
   *Why it matters:* it's an AI-tool integration project, not just a scorer.

2. **6 tools (`score_risk`, wallet, token, contract, trace, chains), 1 resource (scoring method) and 1 prompt (`investigate_address`).**
   *Why it matters:* know the surface area.

3. **Data: Etherscan V2, GoPlus, DexScreener, public RPC, and a small hand-checked local bad-address list.**
   *Why it matters:* multiple sources, and each can fail independently.

4. **Read-only by design: no keys, no signing, `readOnlyHint`, and an RPC method allow-list.**
   *Why it matters:* safe to hand to an AI.

5. **Analysis emits findings (ID, severity, reason, source). A separate pure scorer makes the number.**
   *Why it matters:* the core design decision, testable and explainable.

6. **85 rules: group max (no double counting), clamp to 0–100, 11 decisive findings set a floor of 75, 4 trust signals subtract.**
   *Why it matters:* know how the number is built.

7. **Confidence from source success (all OK = high, ≤50% failed = medium). Missing data lowers confidence, not the score.**
   *Why it matters:* "no data" never reads as "safe".

8. **Contract checks: verified source, EIP-1967 proxy slots, owner type, and a PUSH4 selector scan for unverified bytecode.**
   *Why it matters:* hidden owner powers are the main contract risk.

9. **Tracing is a sample: 100 txs at the root, 50 at hop 2, fan-out 3, never expanding exchanges, protocols or burn addresses.**
   *Why it matters:* it explains the limits of "clean" results.

10. **170 tests pass. The eval set has 34 addresses (12 risky, 22 safe): ROC AUC 0.981, 1 false alarm, 1 miss (0.958 without the local list). Arc Safe Send is live.**
    *Why it matters:* be ready to say what's proven and what isn't.
