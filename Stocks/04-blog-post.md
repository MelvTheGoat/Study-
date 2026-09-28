# Building the measuring stick before the agent: a stock question-answering project

Repo: https://github.com/MelvTheGoat/Stocks

## Why I built it

Most language-model portfolio projects show a few good examples. That proves very little. A model can write a fluent answer about a company and still invent the share price in the middle of it. If you only look at the examples you picked, you'll never see it.

I wanted to build something that answers factual questions about stocks, like prices, market caps and returns, and I wanted to **know** how often it's right. Not guess, not show three nice screenshots.

So this project is built the other way round: the evaluation comes first, and the agent comes second.

There's a second reason. The project covers US stocks and Nigerian stocks listed on the Nigerian Exchange (NGX). Large models have read a huge amount about Apple and very little about Nigerian Breweries or Dangote Cement. That gap is the interesting part. It lets me measure how much of a model's apparent skill is memory rather than reasoning, and whether giving it real data closes the gap.

## Where it stands, honestly

This is early. Here's the status straight from the README:

| Part | State |
|---|---|
| Repo, config, model client, CI | in progress |
| NGX daily price collector | in progress (the folder is empty) |
| Data pipeline (US + NGX) | not started |
| Eval set | not started |
| Kaggle runner | not started |
| Baselines | not run yet |
| Agent | not started |
| Experiments | not run yet |

There are no results yet, and the README says "not run yet" rather than showing a placeholder number. This post is about the foundations and why they're shaped the way they are.

I also started over once. The first version, a week earlier, was a Next.js web app with a small retrieval setup, guardrails and a naira formatter. On 27 September I deleted it and restarted in Python, because the real goal is an agent plus a serious eval that runs on a GPU. That's a Python world.

## The problem, broken down

To measure an agent properly I need:

1. **A run definition that can't lie.** If a result says "temperature 0", it must have been temperature 0.
2. **A model client that's cheap to re-run.** I have a free Kaggle GPU with roughly 30 hours a week. Re-grading shouldn't cost any model calls.
3. **Retries that don't waste the budget.** Retry what might succeed next time, and never retry what won't.
4. **A record of every call**: tokens, latency and whether it was cached, because cost and speed are results too.
5. **Tests that never touch a GPU or the network.**
6. **Data I'm allowed to use.**

The first five are built. The sixth is the hard part.

## How it works

### One YAML file per run, and typos fail

Every run is one YAML file: which model, which sampling settings, which eval split, which kind of agent, which seed. It's loaded into Pydantic models that **reject unknown keys**:

```python
# Rejecting unknown keys is the whole point, so every model in this file shares
# these settings.
_STRICT = ConfigDict(extra="forbid")
```

Why does that matter? If I type `temparature: 0.7`, a loose loader would ignore it. The real temperature stays at the default while the file claims otherwise, and every number from that run is quietly mislabelled. That kind of error survives all the way into a report.

Each config also has a fingerprint, a short SHA-256 of its contents. If two results disagree, the first question is "same config?", and the fingerprint answers it without a diff.

### A stack of small clients

Everything that talks to a model goes through one interface: `chat(request) -> response`. On top of the real HTTP client I stack three wrappers:

```
logging -> cache -> retries -> HTTP
```

The order is deliberate:
- **Retries sit innermost.** A call that fails twice and then succeeds is cached once.
- **The cache sits in the middle.** A repeated request returns the saved answer instantly.
- **Logging sits outermost.** A cache hit is still logged as a question answered, just with zero GPU time. Counting only misses would make a re-run look free and a first run look expensive.

### A cache key that can't go stale

The cache key is a hash of the **entire** request:

```python
def cache_key(self) -> str:
    canonical = json.dumps(
        dataclasses.asdict(self), sort_keys=True, separators=(",", ":"), default=str
    )
    return hashlib.sha256(canonical.encode()).hexdigest()
```

I built it from `dataclasses.asdict` rather than a hand-written list of fields. If someone later adds a new sampling setting to the request, it's part of the key automatically. With a hand-written key, the day someone forgets to add it, the cache starts serving answers made under different settings, and no test would notice.

Cache files are written to a temp file and then renamed. Kaggle kills sessions at the time limit, and a rename is atomic, so the cache can never hold half a file. If it somehow finds a broken file, it treats it as a miss.

### Retry only what might work

The HTTP client sorts every failure into one of two types:

```python
# 429 means slow down, 5xx means the server is unwell or still warming
# up. Everything else in the 4xx range is our own mistake.
if response.status_code == 429 or response.status_code >= 500:
    raise TransientModelError(f"HTTP {response.status_code}: {detail}")
raise PermanentModelError(f"HTTP {response.status_code}: {detail}")
```

Timeouts and "connection refused" (which happens while vLLM is still loading weights) are transient. A 200 response whose body isn't a chat completion is permanent: something else is listening on the port, and asking again gets the same nonsense.

The retry wrapper uses exponential backoff with jitter:

```python
ceiling = min(self.base_delay_s * (2**attempt), self.max_delay_s)
return ceiling * (0.5 + 0.5 * self.rng.random())
```

The jitter matters. If many requests fail at the same moment and all retry after exactly two seconds, they hit the overloaded server at the same moment again.

### Tests that can prove the model *wasn't* called

There are two fake clients. `FakeModelClient` returns scripted replies. It can match on part of the user's message, which matters for agent loops where the order of calls is exactly what's under test. When it runs out of replies it raises an error rather than returning something bland, because a silent fallback would turn "the agent took an extra step" into a passing test.

`NeverCalledClient` fails the test if anything calls it. That's how you prove a cached answer, or a refused question, never reached the GPU. An assertion on the output can't tell you that.

CI runs the whole suite twice, the second time with sockets disabled. If a test quietly starts calling a real service, it fails in CI, not on the day that service is down. Right now 64 tests pass in about a second and a half.

## The hard part: data I'm allowed to use

I wrote down a rule before collecting anything: every source goes into `DATA_SOURCES.md` with the date its terms were checked. No paid data, and no source whose terms forbid automated collection.

The NGX's own website turned out to be the problem. Every page, including the price list and all the terms pages, answers with a JavaScript bot challenge from a Sucuri firewall. A plain HTTP client never gets through, and a GitHub Actions runner would be treated even more harshly. I couldn't read the terms because they're behind the same wall, and there's no robots.txt.

Getting past it would mean running the anti-bot script or replaying its cookie. That's working around a barrier the site put there on purpose, so it's not on the table.

The NGX data portal needs a paid account, which is out of scope. A third-party site, african-markets.com, allows the relevant pages in its robots.txt and serves structured listings, but it has no terms page. A missing terms page isn't the same as permission, so that needs a human decision. Any data from there would also have to be labelled as second-hand.

For US data, the SEC's EDGAR requires a contact email in the User-Agent header. A plain request gets a 403, which is their documented behaviour, not a block.

## What I learned

- **Build the ruler first.** Without an eval, "it looks right" is the only evidence, and it's weak.
- **Make wrong configurations impossible**, not just unlikely. Strict schemas and full-request hashes remove whole classes of silent errors.
- **Retry policy is a cost decision.** On a limited GPU budget, retrying the wrong thing is expensive.
- **Log everything, including cache hits.** Honest cost numbers depend on it.
- **Data access can be the whole project.** Checking terms before writing a scraper saved me from building on something I couldn't use.

## What's next

1. Settle an NGX data source (or narrow the NGX scope), and add US prices plus SEC EDGAR.
2. Build the eval set: dev, test and a hand-checked hard split, frozen at an as-of date, with numeric, exact and source graders.
3. Build the Kaggle runner: vLLM on a T4 with a small quantised model (Qwen2.5-7B-Instruct-AWQ is the current candidate).
4. Run the baselines, closed book and retrieval, and publish the numbers.
5. Build the agent with tools, and measure whether a self-check pass is worth its cost.

Until those runs exist, the honest answer to "how accurate is it?" is: **not measured yet.**
