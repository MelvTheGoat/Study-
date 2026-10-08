# Crypto Risk Checker for AI Assistants: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

---

## Okay, so what are we building?

Alright. Crypto scams are brutal. Send money to a scam token and it's gone. No bank to call, no undo button.

And more and more people now ask AI assistants, "Is this token safe?" But without real data, the assistant can only guess. A confident wrong guess could cost someone their savings.

So here's our project: a small tool pack that any compatible AI assistant can plug into. You give it a crypto address, and it checks it with real data. It returns a risk score from 0 to 100, with a plain reason for every single point.

And one rule above all: it can only *read*. It can never move money.

## So what do we need?

1. **A list of blockchains** we support, and a way to check addresses are valid.
2. **Settings**, with secret keys kept out of the code.
3. **Safe, polite connections** to free data services.
4. **One connection per data service**: Etherscan, GoPlus, DexScreener, and the blockchain itself.
5. **A list of known bad addresses.**
6. **Checks** for wallets, tokens and contracts, and a way to follow money.
7. **A scorer** that turns what the checks found into a number, with reasons.
8. **The MCP server**, so assistants can use it all.
9. **A way to test how accurate it is.**
10. **Automatic checks and packaging.**

The order: basics and settings first, then data connections, then checks, then the scorer, then the server. The checks and the scorer are kept apart on purpose. You'll see why.

## Step zero: set up the workshop

First the basics. A `README.md`, the front page. A `LICENSE`, the MIT licence, which lets anyone use the code. A `.gitignore`, so git doesn't save private files.

`.python-version` says which Python to use: 3.11. `pyproject.toml` describes the project and what it needs. And `uv.lock` pins the exact version of every tool, so everyone installs exactly the same thing.

`.env.example` is a template for settings and secret keys, with explanations. You copy it to `.env`, fill in your keys, and the private copy is never saved.

The code lives in `src/web3_risk_mcp/`. Its `__init__.py` describes it in one line: "a read-only MCP server that checks wallets, tokens, and contracts for risk."

`src/web3_risk_mcp/config.py` loads the settings from those environment variables or the `.env` file. And `src/web3_risk_mcp/errors.py` defines error messages "written for people, not machines", so when something fails, you get a sentence you can understand.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `LICENSE`: the MIT licence.
> - `.gitignore`: files git should not save.
> - `.python-version`: which Python to use.
> - `pyproject.toml`: the project's details and needs.
> - `uv.lock`: exact versions of every tool.
> - `.env.example`: a template for settings and keys.
> - `src/web3_risk_mcp/__init__.py`: marks the main code folder as a package.
> - `src/web3_risk_mcp/config.py`: loads the settings.
> - `src/web3_risk_mcp/errors.py`: error messages people can understand.

## Step one: which blockchains, and is this address real?

`src/web3_risk_mcp/chains.py` lists the blockchains we support: Ethereum, Base, Arbitrum One, Polygon PoS and BNB Chain. Later we'll add a sixth, Arc. They're all "EVM" chains, meaning they work the same way, so one set of checks covers them all.

It also checks an address is written correctly, "0x" followed by the right characters, before we waste any calls on it.

Tested in `tests/test_chains.py`. And `tests/test_config.py` checks the settings load properly.

> **📁 Files we just created**
> - `src/web3_risk_mcp/chains.py`: the supported blockchains, and address checks.
> - `tests/test_chains.py` and `tests/test_config.py`: tests for chains and settings.

## Step two: safe connections to free services

Now the data. We use free services, and free services are slow and have limits. So before anything else, we build one shared, careful layer for talking to them all: `src/web3_risk_mcp/clients/http.py`.

It gives every service four things. A cache, so the same question asked twice within 5 minutes doesn't call the service twice. A rate limiter, so we stay just under each free plan's limit.

Retries that wait longer each time, with a little random wobble so lots of retries don't hit at once. And key hiding: secret keys are stripped from logs, saved answers and error messages, so they can never leak.

Then one file per service, each using that shared layer:

- `clients/etherscan.py`: Etherscan, a "block explorer" that indexes everything on a chain. It gives transaction history and a contract's published code. One key covers all the chains.
- `clients/goplus.py`: GoPlus, which runs automatic security checks. It flags honeypots, buy and sell taxes, owner powers and scam labels.
- `clients/dexscreener.py`: DexScreener, which tracks trading pools on decentralised exchanges, where people swap tokens without a company in the middle. It tells us how much money is in a token's pools.
- `clients/rpc.py`: a direct line to the blockchain itself, for questions like "what's this balance?" or "what code lives here?"

## The second safety net

Now here's an important design choice in `rpc.py`. It has a short list of allowed "read" commands. Anything else is refused. Here's the real line:

```python
raise SourceError(NAME, f"Method {method} is not allowed. This server is read-only.")
```

So even if some code somewhere had a bug, it could never send a transaction. It's a second safety net, on top of every tool being marked read-only.

`src/web3_risk_mcp/services.py` builds all these connections when the server starts, sharing one connection pool, and closes them when it stops.

Tests: `tests/test_http.py` checks the cache, limits, retries and key hiding. `tests/test_clients.py` checks each service connection. And `tests/mocks.py` fakes every service, so tests never touch the internet.

> **📁 Files we just created**
> - `src/web3_risk_mcp/clients/http.py`: the shared safe layer: cache, limits, retries, key hiding.
> - `src/web3_risk_mcp/clients/etherscan.py`: history and contract code.
> - `src/web3_risk_mcp/clients/goplus.py`: security flags.
> - `src/web3_risk_mcp/clients/dexscreener.py`: trading pool data.
> - `src/web3_risk_mcp/clients/rpc.py`: direct blockchain questions, read-only by an allow-list.
> - `src/web3_risk_mcp/clients/__init__.py`: marks the clients folder as a package.
> - `src/web3_risk_mcp/services.py`: builds all the connections at start-up.
> - `tests/test_http.py`, `tests/test_clients.py`, `tests/mocks.py`: connection tests and fake services.

## Step three: a list of known bad actors

GoPlus covers a lot, but it's good to have a few certain matches of our own. So `src/web3_risk_mcp/data/known_addresses.json` is a small, hand-checked list of well-known addresses: mixers, which hide where money came from, and known thieves' wallets. Each was checked against public explorer labels and incident reports.

As the file itself says, it's "a starting point, not a full blocklist". `src/web3_risk_mcp/labels.py` looks addresses up in it.

> **📁 Files we just created**
> - `src/web3_risk_mcp/data/known_addresses.json`: a hand-checked list of known bad addresses.
> - `src/web3_risk_mcp/data/__init__.py`: marks the data folder as a package.
> - `src/web3_risk_mcp/labels.py`: looks addresses up in that list.

## Step four: one shape for everything we find

Before writing checks, we decide what they report. That's `src/web3_risk_mcp/models.py`.

Every report says which chain and address it's about, which data sources answered, and a list of **findings**. A finding is one thing a check noticed, like "the owner can create unlimited new tokens". It has a fixed ID, a severity, a plain reason, and the source it came from.

And here's the key idea: checks only *describe*. They never give scores. Keep that in mind, because it's what makes the scorer simple and fair.

`src/web3_risk_mcp/analysis/common.py` has helpers every check shares. The most important wraps each call to a service and records whether it worked. So if GoPlus is down, the check carries on and simply notes the gap.

> **📁 Files we just created**
> - `src/web3_risk_mcp/models.py`: one shape for every report and finding.
> - `src/web3_risk_mcp/analysis/common.py`: shared helpers, including recording which sources answered.
> - `src/web3_risk_mcp/analysis/__init__.py`: marks the analysis folder as a package.

## Step five: the checks

Now the checks themselves, each in its own file in `src/web3_risk_mcp/analysis/`.

**`address.py`** asks one question: is this address on a list of known bad actors, ours or GoPlus's?

**`wallet.py`** profiles a wallet. How old is it, how much does it hold, who does it deal with most, who first sent it money, and how does it behave?

**`token.py`** checks a token. Can you actually sell it, or is it a honeypot? Are there buy and sell taxes? Can the owner create new tokens, block holders, or freeze trading?

Do a few wallets own most of it? Is there real money in its pools, and is it locked?

**`contract.py`** inspects a contract. Is its code public? Can it be changed later, through what's called a proxy?

Who controls it: one person, a group needing several approvals, or nobody? And which functions could hurt users? It ends with a plain-English summary.

But what if a contract's code isn't public? That's where **`selectors.py`** comes in. Every function in a contract has a short fingerprint. This file has a catalogue of risky functions' fingerprints, like "mint", and scans the contract's raw code for them.

**`trace.py`** follows the money. Hop 1 is everyone who sent money directly to the address, or received it.

Hop 2 goes one step further, for the 3 busiest partners. It skips exchanges and big services, which would link to everyone. And it flags any link to mixers, sanctioned addresses or thieves, showing the path.

Tests: `tests/test_wallet.py`, `tests/test_token.py`, `tests/test_contract.py` and `tests/test_trace.py`.

> **📁 Files we just created**
> - `src/web3_risk_mcp/analysis/address.py`: is it a known bad actor?
> - `src/web3_risk_mcp/analysis/wallet.py`: a wallet's profile and behaviour.
> - `src/web3_risk_mcp/analysis/token.py`: honeypots, taxes, owner powers, liquidity.
> - `src/web3_risk_mcp/analysis/contract.py`: public code, upgradeability, who's in control.
> - `src/web3_risk_mcp/analysis/selectors.py`: finds risky functions in raw contract code.
> - `src/web3_risk_mcp/analysis/trace.py`: follows money for one or two hops.
> - `tests/test_wallet.py`, `tests/test_token.py`, `tests/test_contract.py`, `tests/test_trace.py`: check tests.

Let's pause and look at where we are. We have safe connections, a list of known bad actors, one shape for findings, and six checks that describe what they see. But we still don't have a number. That's next.

## Step six: the scorer

`src/web3_risk_mcp/scoring.py` turns findings into a score. And it's completely transparent. Anyone could redo it by hand.

It works in four steps. Every finding has a fixed ID, like `token.honeypot`. A rule table gives each ID its points, 92 rules in all by now.

Findings about the same problem share a group, and only the biggest one counts, so nothing is counted twice. Owner powers, like "can mint" or "can freeze", add at most 30 points together, because regulated coins like USDC have all of them and aren't scams. Then the total is kept between 0 and 100.

And then the deal-breakers. 12 findings are "decisive", like a honeypot or a sanctioned address. Here's the real number:

```python
DECISIVE_FLOOR = 75
```

If any decisive finding shows up, the score can't be below 75, which is "critical". A deal-breaker can never be outvoted.

Two more things. Trusted things get points *taken off*, like well-known tokens and long-established wallets.

And confidence is based on how many data sources answered. If sources fail, confidence drops, but the score is never lowered. So missing data can never make an address look safer.

Then `src/web3_risk_mcp/analysis/score.py` ties it together, for the main `score_risk` tool. It asks the blockchain whether the address has code. No code means a wallet, so it runs the wallet check and a one-hop trace.

Code means a contract or token, so it runs the token, contract and address checks. They all run at the same time, then everything goes to the scorer.

Tests: `tests/test_scoring.py` and `tests/test_score_risk.py`.

> **📁 Files we just created**
> - `src/web3_risk_mcp/scoring.py`: 92 rules, no double counting, the 75 floor, and confidence.
> - `src/web3_risk_mcp/analysis/score.py`: runs the right checks, then scores them.
> - `tests/test_scoring.py` and `tests/test_score_risk.py`: scorer tests.

## Step seven: documents that can't go stale

Here's a lovely idea. The scoring rules need explaining, but documents usually go out of date.

So `src/web3_risk_mcp/method.py` builds the explanation document straight from the live rule table. `scripts/render_method_doc.py` writes it to `docs/risk-method.md`. And a test fails if the document and the code ever differ. So it can never drift out of date.

> **📁 Files we just created**
> - `src/web3_risk_mcp/method.py`: builds the method document from the rule table.
> - `scripts/render_method_doc.py`: writes that document.
> - `docs/risk-method.md`: the scoring method, always matching the code.

## Step eight: the MCP server

Now we make it all usable by AI assistants. MCP, the Model Context Protocol, is a standard way for assistants to use outside tools, like a universal plug socket.

`src/web3_risk_mcp/server.py` tells assistants what's available. There are 6 tools: score the risk, profile a wallet, check a token, inspect a contract, follow the money, and list supported chains. Every tool is marked read-only. There's 1 resource, the scoring method document, so the assistant can explain how scores work.

And there's 1 prompt, in `src/web3_risk_mcp/prompts.py`. A prompt is a ready-made set of instructions the user can pick. This one walks the assistant through a proper investigation, step by step, and explains results simply.

`src/web3_risk_mcp/__main__.py` starts the server. It can run in two ways: a direct link on your own computer, for desktop apps and code editors, or over the web, for remote or shared use.

Tested in `tests/test_server.py`, with shared test helpers in `tests/conftest.py` and `tests/__init__.py`.

> **📁 Files we just created**
> - `src/web3_risk_mcp/server.py`: the 6 tools, 1 resource and 1 prompt.
> - `src/web3_risk_mcp/prompts.py`: a step-by-step investigation guide for the assistant.
> - `src/web3_risk_mcp/__main__.py`: starts the server, locally or over the web.
> - `tests/test_server.py`, `tests/conftest.py`, `tests/__init__.py`: server tests and shared helpers.

## Step nine: how accurate is it?

Now the honest question: does the score actually separate risky from safe?

`eval/dataset.json` holds 34 hand-labelled addresses: 12 risky and 22 safe. Each label says where it came from. It's small on purpose, so every label could be checked by hand.

`src/web3_risk_mcp/evaluation.py` has the measuring tools. Precision is how many flagged addresses were truly risky. Recall is how many risky ones were caught. And ROC AUC measures how well the scores separate the two groups overall.

It also has a "cassette": it records real service answers to a file once, then replays them. So the evaluation can be re-run offline and give the same numbers.

`eval/run_eval.py` runs it all. And it runs twice: once with our own bad-address list, and once without, to show what works without it.

Tested in `tests/test_evaluation.py`.

> **📁 Files we just created**
> - `eval/dataset.json`: 34 hand-labelled addresses.
> - `src/web3_risk_mcp/evaluation.py`: precision, recall, ROC AUC, and the record-and-replay cassette.
> - `eval/run_eval.py`: runs the evaluation, with and without the local list.
> - `tests/test_evaluation.py`: tests the measuring tools.

When we run it for real, the results go into `eval/results.md` and `eval/results.json`, and the recorded answers into `eval/fixtures/`. The first run, on 30 September, gave a ROC AUC of 0.981. It caught 11 of the 12 risky addresses, and wrongly flagged 1 of the 22 safe ones.

> **📁 Files we just created**
> - `eval/results.md` and `eval/results.json`: the first real results.
> - `eval/fixtures/`: the recorded answers, so anyone can replay it offline.

## Step ten: automatic checks and packaging

`.github/workflows/ci.yml` checks every change automatically. It runs the linter and the format check, runs the tests on three Python versions, and builds the Docker box.

`Dockerfile` packs the server into a box called a container that runs the same anywhere. The box doesn't run with full admin rights, which is safer. `.dockerignore` says what to leave out of it.

> **📁 Files we just created**
> - `.github/workflows/ci.yml`: lint, format, tests on 3 Python versions, and a Docker build.
> - `Dockerfile`: packs the server into a box.
> - `.dockerignore`: what to leave out of the box.

We also get it ready for PyPI, the public shelf of Python packages, so people could install it with one command. A release workflow, `.github/workflows/release.yml`, publishes a new version when it's tagged. On 8 October, PyPI didn't list it yet.

## Step eleven: Arc Safe Send

In October we add a sixth blockchain: Arc, made by Circle, the company behind USDC. On Arc, USDC is the main coin. That brings three new problems.

First, Circle keeps a blocklist. A payment to a blocked address fails, but you still pay the fee. So every Arc check now reads the USDC and EURC blocklists, and a blocked address is a deal-breaker.

Second, we want to know if a payment would go through before anyone sends it. So we simulate a 1 USDC payment with a read-only call. Nothing is sent or signed.

Third, USDC on Arc is logged twice, at two different decimal sizes. So we read money moves only from one system log, and nothing gets counted twice. `tests/test_arc.py` checks all of this by replaying 70 recorded answers from the real Arc network.

Then we build a web page for people, not just assistants: Arc Safe Send. `src/web3_risk_mcp/web/` holds the small FastAPI app and its page. It caches answers, and limits how many checks each visitor can run, so the free API keys don't run out. `Dockerfile.web` and `render.yaml` put it online for free on Render, at https://arc-safe-send.onrender.com.

After a check, you can pay from your own browser wallet. The server never sees a key. And `contracts/` holds RiskAttestation, a tiny contract that saves a check on Arc for anyone to see: the address, the score, the rule version and a fingerprint of the findings. It's now deployed on Arc mainnet.

> **📁 Files we just created**
> - `src/web3_risk_mcp/analysis/arc.py`: the Arc checks: blocklists, the payment test and the one USDC log.
> - `tests/test_arc.py`: Arc checks, replaying real Arc answers.
> - `src/web3_risk_mcp/web/`: the Arc Safe Send app and page.
> - `Dockerfile.web` and `render.yaml`: put the web app online.
> - `src/web3_risk_mcp/attestation.py`: builds the "save this check" call.
> - `contracts/`: the RiskAttestation contract, its tests and a deploy script.
> - `tests/test_web.py` and `tests/test_attestation.py`: tests for the web app and saved checks.
> - `.github/workflows/release.yml`: publishes new versions to PyPI.

## So, how's it doing?

170 tests pass in about 10 seconds, with every outside service faked. The contract has 8 more tests of its own.

On the 34 test addresses, the ROC AUC was 0.981. It caught 11 of 12 risky ones and gave 1 false alarm in 22 safe ones. With our own bad-address list switched off, it still caught 9 of 12.

## What's still missing?

- **The points per rule were set by hand**, based on known scam patterns.
- **34 test addresses is small**, so the numbers are rough, and none of them are on Arc yet.
- **It only looks at recent history**: the latest 100 transactions, and the busiest paths.
- **A brand-new scam** that services haven't seen can score low. Low means "no red flags found", not "safe".
- **Etherscan's free plan** has no account history on Base or BNB Chain.

## Let's put it all together

So let's look at it in one breath.

We **set up the workshop** with pinned versions and private keys. We listed **the blockchains** and checked addresses. We built **one safe layer** for every data service, with a cache, limits, retries and key hiding, and **one connection per service**, with a read-only allow-list for the blockchain.

We added a **list of known bad actors**, decided **one shape for findings**, and wrote **six checks** that only describe what they see. Then a **transparent scorer**: 92 rules, no double counting, a floor of 75 for deal-breakers, and confidence that drops when data is missing.

We made the **method document build itself** from the code, wrapped it all in an **MCP server**, built an **honest evaluation** with replayable answers, and added **automatic checks**. Then we added **Arc**, with blocklist checks, a payment test, a live web page and a contract that saves checks on the blockchain.

Notice how it links. Keeping checks and scoring apart is what makes every point explainable. The read-only rule shows up twice, in the tools and in the blockchain connection. And recording which sources answered, back in step four, is what powers the confidence level in step six.

That's the project. Real data, a reason for every point, and no way to move money.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
