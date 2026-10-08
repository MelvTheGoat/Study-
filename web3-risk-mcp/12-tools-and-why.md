# Crypto Risk Checker for AI Assistants: Tools and Why They Were Used

This file covers every tool and data service the project uses. For each one you get **what it is**, in plain English, and **why this project uses it**. The ideas behind them, like MCP or rate limiting, are explained in [11-technical-terms.md](11-technical-terms.md).

All the data services are used on their free plans. Everything the tool does is read-only.

---

## The Language

### Python 3.11+
**What it is:** a popular programming language known for being easy to read. "3.11+" means version 3.11 or newer.

**Why it's used here:** it has good support for doing many things at once (called "async"), which matters because each check calls several services at the same time.

---

## Talking to AI Assistants

### MCP Python SDK
**What it is:** the official Python toolkit for building MCP servers. MCP is the standard way AI assistants use outside tools.

**Why it's used here:** it handles all the protocol details. The project just describes its 6 tools, 1 resource and 1 prompt, and the SDK makes them available to any compatible assistant.

---

## Fetching Data

### httpx
**What it is:** a Python tool for sending requests to websites and APIs. It can wait for many answers at once.

**Why it's used here:** every call to an outside service goes through it, with time limits. The project's own layer adds a 5-minute cache, rate limits, retries and key hiding on top.

### Etherscan (V2 API)
**What it is:** a popular service for looking up blockchain data: transaction history and published contract code.

**Why it's used here:** it provides wallet history and a contract's published source code. One free key covers all 6 supported blockchains, including Arc. The free plan doesn't include account history on Base or BNB Chain.

### GoPlus
**What it is:** a security service that scans tokens and addresses for known scam signs.

**Why it's used here:** it gives broad coverage: honeypot tests, buy and sell taxes, owner powers and scam labels. Most token checks rely on it.

### DexScreener
**What it is:** a service that tracks trading pools on decentralised exchanges (places where people swap tokens directly).

**Why it's used here:** it shows how much money is in each token's pools, which reveals rug pull risk.

### Public RPC
**What it is:** free, open connections for asking a blockchain questions directly.

**Why it's used here:** it checks balances, whether an address has code, and which code a proxy contract points to. The project's RPC part refuses anything outside a short list of "read" commands.

### Local list of known bad addresses
**What it is:** a small, hand-checked file of known mixers and thieves' addresses, kept inside the project.

**Why it's used here:** it gives highly reliable matches for well-known bad actors. The evaluation runs both with and without it, to show what works without it.

---

## Settings and Data Shapes

### pydantic and pydantic-settings
**What it is:** pydantic checks data has the right shape and types. pydantic-settings loads settings, like API keys, from a private `.env` file.

**Why it's used here:** results come back in a clear, typed shape, and keys stay out of the code.

### pycryptodome
**What it is:** a Python tool for cryptography, the maths behind secure codes and fingerprints.

**Why it's used here:** it makes Keccak fingerprints, the kind Ethereum uses. These are needed to find function fingerprints in raw contract code, and to find where a proxy stores its settings.

---

## Packaging, Testing and Running

### uv
**What it is:** a fast tool for installing Python packages and locking their exact versions.

**Why it's used here:** a lock file means everyone installs exactly the same versions, so the tool behaves the same everywhere.

### pytest, pytest-asyncio and respx
**What it is:** pytest runs tests. pytest-asyncio lets it test code that does many things at once. respx fakes web services during tests.

**Why it's used here:** 170 tests pass in about 10 seconds. Every outside service is faked, so no keys or internet are needed.

### ruff
**What it is:** a fast "linter" and formatter: it flags mistakes and tidies the code's layout.

**Why it's used here:** it keeps the code consistent and catches small errors early.

### Docker
**What it is:** a tool that packs an app and everything it needs into one box, called a container, that runs the same anywhere.

**Why it's used here:** it lets you run the server anywhere, using either connection type. The container doesn't run with full admin rights, which is safer.

### GitHub Actions
**What it is:** a free service from GitHub that runs checks automatically every time the code changes.

**Why it's used here:** it runs the linter, the format check and the tests on three Python versions (3.11, 3.12 and 3.13), then checks the Docker box builds.

---

## Quick Summary

| Tool | Job in one line |
|---|---|
| Python 3.11+ | The language, with async for parallel checks |
| MCP SDK | Makes the tools usable by AI assistants |
| httpx | Sends requests to outside services |
| Etherscan | History and contract code |
| GoPlus | Scam flags for tokens and addresses |
| DexScreener | Trading pool and liquidity data |
| Public RPC | Direct, read-only blockchain questions |
| Local bad-address list | Reliable matches for known bad actors |
| pydantic | Typed results and private settings |
| pycryptodome | Ethereum-style fingerprints |
| uv | Exact, repeatable installs |
| pytest + respx | Tests with faked services |
| ruff | Tidy, consistent code |
| Docker | Run it anywhere |
| GitHub Actions | Checks every change automatically |
