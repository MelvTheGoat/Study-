# Crypto Risk Checker for AI Assistants: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms are grouped from the ground up: crypto basics, the scams being checked for, how the AI connects, how data is fetched, and how the score is built.

---

## 1. Crypto Basics

### Blockchain
**What it means:** a shared public record of every transaction, copied across many computers so nobody can secretly change it.

**Why it's needed here:** everything the tool checks lives on a blockchain. Because it's public, anyone can read an address's history.

### EVM (Ethereum Virtual Machine)
**What it means:** the system that runs programs on Ethereum. Many other blockchains copy it.

**Why it's needed here:** the tool supports 6 EVM blockchains: Ethereum, Base, Arbitrum One, Polygon PoS, BNB Chain and Arc. Because they work the same way, one set of checks covers them all.

### Address
**What it means:** an ID on the blockchain, like an account number, written as "0x" followed by 40 characters.

**Why it's needed here:** it's what the user asks about. The tool first checks the address is in a valid format.

### Wallet
**What it means:** an address controlled by a person with a secret key. It holds no code.

**Why it's needed here:** wallets get different checks from contracts: age, behaviour, who sent them money first, and where their money went.

### Smart contract
**What it means:** a program that lives on the blockchain and runs by itself.

**Why it's needed here:** contracts can hide dangerous powers, like letting the owner freeze your coins. If an address has code, the tool treats it as a contract or token.

### Token
**What it means:** a digital coin created by a smart contract.

**Why it's needed here:** most everyday scams are token scams. Anyone can make one in minutes.

### Transaction
**What it means:** one transfer or action recorded on the blockchain.

**Why it's needed here:** a wallet's transactions show its behaviour, and they're the path used to follow money.

### Liquidity and liquidity pool
**What it means:** a liquidity pool is a pot of two coins that lets people swap one for the other. Liquidity is how much money is in it.

**Why it's needed here:** a token with tiny liquidity is hard to sell. If liquidity can be pulled out suddenly, it's a classic scam setup.

### Verified contract
**What it means:** a contract whose readable source code has been published and matched to what's on the blockchain.

**Why it's needed here:** verified code can be read and checked. Unverified code is a warning sign, so the tool inspects its raw code instead.

---

## 2. Scams Being Checked For

### Honeypot
**What it means:** a token you can buy but can never sell.

**Why it's needed here:** it's one of the most common scams. It's a "decisive" finding, which pushes the score to at least 75.

### Rug pull
**What it means:** when the creators suddenly pull out all the money from a token's pool, leaving buyers with worthless coins.

**Why it's needed here:** the tool checks liquidity and whether it's locked, which are the warning signs.

### Owner powers (mint, blacklist, pause)
**What it means:** special abilities a contract's owner might keep. Mint creates new tokens, blacklist blocks certain holders, and pause freezes trading.

**Why it's needed here:** these powers can be used to steal value or trap buyers. Hidden owner powers are the main risk in contracts.

### Buy and sell tax
**What it means:** a fee the token takes every time you buy or sell.

**Why it's needed here:** a very high sell tax can quietly take most of your money when you try to leave.

### Holder concentration
**What it means:** how much of a token is owned by just a few wallets.

**Why it's needed here:** if one wallet owns most of it, it can crash the price by selling.

### Proxy (upgradeable contract)
**What it means:** a contract that points to another contract for its actual code. That code can be swapped later.

**Why it's needed here:** if a single person can swap the code, the contract can change its rules after you trust it.

### Multisig
**What it means:** a wallet that needs several people to approve each action.

**Why it's needed here:** a contract controlled by a multisig is safer than one controlled by a single person. The tool reports who's in control.

### Mixer
**What it means:** a service that blends many people's coins to hide where money came from.

**Why it's needed here:** stolen money often goes through mixers. A link to one is a strong warning sign when following money.

### Sanctioned address
**What it means:** an address officially banned by governments.

**Why it's needed here:** receiving money from one is serious. A direct link counts as critical, and a link two steps away counts as high.

---

## 3. Connecting to the AI

### MCP (Model Context Protocol)
**What it means:** a standard way for AI assistants to use outside tools. Like a universal plug socket for AI.

**Why it's needed here:** write the tools once, and any MCP-compatible assistant can use them.

### MCP server
**What it means:** a program that offers tools to AI assistants using MCP.

**Why it's needed here:** it's what this project is. It offers 6 tools, 1 resource (the scoring method) and 1 prompt.

### Tool
**What it means:** one action the assistant can ask for, like "check this token".

**Why it's needed here:** the main tool is `score_risk`. Others check a wallet, a token, or a contract, follow money, or list the supported blockchains.

### Resource
**What it means:** a document the server can hand to the assistant to read.

**Why it's needed here:** the scoring method is shared as a resource, so the assistant can explain how scores work.

### Prompt
**What it means:** a ready-made set of instructions for the assistant.

**Why it's needed here:** it gives the assistant a step-by-step plan for investigating an address, and explains results simply to a beginner.

### stdio and HTTP transport
**What it means:** two ways for the assistant to talk to the server. stdio is a direct link on the same computer. HTTP works over a network.

**Why it's needed here:** desktop apps use stdio, while remote or Docker setups use HTTP.

### Read-only
**What it means:** can look at data but never change anything.

**Why it's needed here:** a risk checker should never be able to move money. Every tool is marked read-only, and the blockchain connection refuses anything outside a short "read" list.

---

## 4. Fetching Data

### API and API key
**What it means:** an API lets one program ask another for data. A key is a secret code that identifies you.

**Why it's needed here:** the tool gets data from Etherscan, GoPlus and DexScreener. Etherscan uses a free key, and one key covers all 6 blockchains. Keys are kept in a private settings file, not in the code.

### RPC (remote procedure call)
**What it means:** a direct way to ask a blockchain node (a computer holding the blockchain) a question.

**Why it's needed here:** it reads balances and code straight from the blockchain, like "does this address have code?"

### Cache
**What it means:** a saved copy of an answer, reused for a while.

**Why it's needed here:** asking about the same address twice is common. Answers are saved for 5 minutes.

### Rate limiting
**What it means:** keeping how often you send requests under a limit.

**Why it's needed here:** free API plans only allow a few requests per second. Each source has its own limit set just under its free plan.

### Retries with backoff and jitter
**What it means:** trying again after a failure, waiting longer each time, with a small random change to each wait.

**Why it's needed here:** busy services often fail for a moment. The randomness stops many retries from hitting at exactly the same time.

### Key redaction
**What it means:** removing secret keys from anything that might be shown or saved.

**Why it's needed here:** keys are stripped from logs, cache entries and error messages, so they can't leak.

### Data gap
**What it means:** information the tool couldn't get, with the reason.

**Why it's needed here:** the answer lists every gap, so a missing check is never hidden. For example, Etherscan's free plan doesn't give account history on Base or BNB Chain.

---

## 5. Building the Score

### Finding
**What it means:** one fact spotted by a check, with an ID, a severity, a reason and a source.

**Why it's needed here:** checks only produce findings. Keeping them separate from scoring makes both simple and testable.

### Severity
**What it means:** how serious a finding is: info, low, medium, high or critical.

**Why it's needed here:** if a finding has no specific rule, its severity sets the points (critical 40, high 15, medium 8, low 2).

### Rule table
**What it means:** a list saying how many points each finding is worth.

**Why it's needed here:** all 92 rules live in one place, so they're easy to read and adjust. A document is generated from it, and a test fails if the two ever differ.

### Grouping (no double counting)
**What it means:** findings about the same problem share a group, and only the biggest one counts.

**Why it's needed here:** two services might both report "honeypot". Counting it twice would unfairly double the score.

### Decisive finding and floor
**What it means:** a decisive finding is a deal-breaker. The floor is a minimum score it sets.

**Why it's needed here:** 11 findings are decisive, like a honeypot. Any one pushes the score to at least 75, so a deal-breaker can't be outvoted.

### Trust signal
**What it means:** a finding that takes points off.

**Why it's needed here:** well-known, trusted tokens and long-established wallets shouldn't look risky. There are 4 of these.

### Clamping
**What it means:** keeping a number within a range.

**Why it's needed here:** the score always stays between 0 and 100, even after trust signals take points off.

### Risk level
**What it means:** a word band for the score: low (under 20), medium (20+), high (50+), critical (75+).

**Why it's needed here:** a word is easier to act on than a number.

### Confidence
**What it means:** how much to trust the score, based on how many data sources answered.

**Why it's needed here:** missing data lowers confidence, never the score. So missing data can't make an address look safer.

### Fund tracing (hops)
**What it means:** following money from one address to the next. Each step is a hop.

**Why it's needed here:** it checks whether an address is linked to mixers, sanctioned wallets or thieves. The tool follows up to 2 hops, skipping exchanges and big services, which would link to everyone.

### Function selector and bytecode scan
**What it means:** bytecode is a contract's raw, machine-readable code. A function selector is a short fingerprint of each function's name inside it.

**Why it's needed here:** when a contract's code isn't published, the tool scans the bytecode for fingerprints of known risky functions, like "mint". Renamed functions can slip through.

---

## 6. Checking the Tool

### Evaluation set
**What it means:** examples with known answers, used to test the tool.

**Why it's needed here:** there are 34 hand-labelled addresses (12 risky, 22 safe). The first live run (30 September 2026) caught 11 of 12 risky ones, with 1 false alarm.

### Precision and recall
**What it means:** precision is how many flagged addresses were truly risky. Recall is how many risky addresses were caught.

**Why it's needed here:** they're the main measures, using a score of 50 as the line between "risky" and "safe". Both came out at 0.917.

### ROC AUC
**What it means:** a number from 0.5 (guessing) to 1.0 (perfect) for how well scores separate risky from safe, across every possible cut-off line.

**Why it's needed here:** it tests the score as a whole, not just at the 50 line. It came out at 0.981.

### Record and replay (cassette)
**What it means:** saving real service answers once, then replaying them in later runs.

**Why it's needed here:** it makes the evaluation repeatable offline. The recorded answers are now saved in the project.

### Mocking
**What it means:** replacing a real service with a pretend one in tests.

**Why it's needed here:** all 170 tests use pretend services, so they need no keys or internet.
