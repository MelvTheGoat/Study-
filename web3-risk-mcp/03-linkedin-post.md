# web3-risk-mcp: LinkedIn Post

*About 160 words. Copy from the line below.*

---

"Is this token safe?" People now ask AI assistants that question. Without data, the assistant can only guess.

So I built web3-risk-mcp: an MCP server that gives any AI assistant real tools to check a crypto wallet, token or smart contract before you touch it.

It checks for honeypots (you can buy but can't sell), hidden owner powers, high sell taxes, unlocked liquidity, and money linked to mixers or known exploiters, across 5 EVM chains.

The part I care about most: every point of the 0–100 score has a reason. The analysis code only describes what it sees. A separate, pure scorer turns that into points using a public rule table. Decisive findings, like a honeypot, set a floor of 75 so "trusted" signals can't hide them.

It's read-only by design. It never asks for a key and can't send a transaction.

102 tests. A 34-address evaluation set is ready, and the live results are next.

https://github.com/MelvTheGoat/web3-risk-mcp

#Web3 #MCP #Python #FraudDetection
