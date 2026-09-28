# Stocks: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Stocks

---

## 60-second pitch

> "I'm building an agent that answers factual questions about US and Nigerian stocks, and I'm building the evaluation first, because a language model can sound confident and still invent a share price. The Nigerian side is deliberate: models know a lot about Apple and very little about Nigerian Breweries, so comparing the two shows how much is memory versus reasoning.
>
> What's built so far is the foundation. One strict YAML file defines each run, so a typo fails instead of mislabelling results. Every model call goes through a stack: logging, a disk cache keyed by a hash of the full request, retries that only fire on errors worth retrying, then an OpenAI-compatible HTTP client for vLLM. There are 64 tests, and CI runs them a second time with the network off.
>
> It's early. There's no eval set or agent yet, and no results. The main blocker is data: the NGX website blocks automated access, and I won't bypass it."

---

## Likely questions and honest answers

### 1. "What does it actually do today?"
Today it's the plumbing: config loading, the model client stack, fakes for testing, and CI. It can't answer a stock question yet. I built the parts that every experiment will depend on first, so the results are trustworthy when they arrive.

### 2. "Why build the eval before the agent?"
Without an eval, the only evidence is examples I picked myself, and a model can look fluent while making numbers up. With a fixed set of questions and known answers, frozen at an as-of date, I can compare the closed-book, retrieval and agent setups fairly, and see whether each change actually helps.

### 3. "Why this cache design?"
The GPU budget is about 30 hours a week on Kaggle. Re-grading or re-running shouldn't cost model calls. The key is a SHA-256 of the whole request, built from `asdict`, so any new field is included automatically. Files are written then renamed, which is atomic, because Kaggle kills sessions at the time limit.

### 4. "How do you decide what to retry?"
Two error types. Transient: timeouts, connection errors, 429 and 5xx. Those might work next time, for example while vLLM is still loading. Permanent: other 4xx, or a 200 with a body that isn't a chat completion. Retrying those wastes GPU time. Backoff is exponential with jitter, capped at 30 seconds, with 4 retries.

### 5. "Why is logging outside the cache?"
So cache hits still count as answered questions, just with zero GPU time. If I only logged misses, a re-run would look free and the first run would look expensive. I also keep a `cached` flag next to latency, so the latency numbers aren't polluted by cache hits.

### 6. "Why a small open model instead of a big API model?"
Cost and control. The free GPU is a T4 with 16 GB. A 7B model at 4-bit is about 5 GB and leaves room for the KV cache. Open weights also make runs reproducible. The honest trade-off is that a small model will be weaker, and part of the experiment is how much tools and data close that gap. Qwen2.5-7B-AWQ is a candidate, not a final choice.

### 7. "How do you know it works?"
For the plumbing: 64 tests, a no-network CI pass, and a `NeverCalledClient` that proves certain paths never reach the model. For the agent's accuracy: **it isn't measured yet**, because the eval and agent don't exist. I won't quote a number I don't have.

### 8. "What would break first?"
Data access. The NGX site is behind a Sucuri bot filter, so I can't collect from it, and the alternative site has no terms page. After that, Kaggle session limits and vLLM start-up time. Those are handled by the cache, atomic writes and transient retries.

### 9. "How would you scale it?"
Batch requests to vLLM (the client is one request at a time today), move the cache to a key-value store, ship the call log to a warehouse, move serving from Kaggle to a paid GPU host or a hosted API, and license proper market data.

### 10. "Why not scrape the NGX site with a headless browser?"
Because the bot filter is there on purpose, and I couldn't read the terms of use. Getting past the challenge is working around a deliberate barrier. I recorded this in `DATA_SOURCES.md` and I'm looking for a permitted source instead.

### 11. "Why Pydantic with `extra=forbid`?"
A typo like `temparature` would otherwise be silently dropped. The run would use the default temperature while the file says otherwise, and the mislabelled result could end up in a report. Strict validation turns that into an immediate error.

### 12. "What's the fingerprint for?"
It's a short hash of the whole config, stored next to results. If two results disagree, I can check straight away whether they came from the same setup.

### 13. "You deleted an earlier version. Why?"
The first version was a small Next.js app with retrieval and guardrails over a tiny text corpus. The real goal is an agent evaluated on a GPU, which is Python territory. I restarted rather than bolting a Python eval onto a TypeScript demo.

### 14. "How will you stop yourself overfitting the eval?"
The config already has `dev`, `test` and `hard` splits. The plan is to tune only on `dev` and run `test` once at the end. The `hard` split is hand-checked document questions.

---

## Weak spots and how to answer them

| Weak spot | Likely poke | Honest answer |
|---|---|---|
| No agent or eval yet | "So there's nothing to show?" | "There's tested infrastructure. The agent and eval are next. I'd rather show that than a demo with made-up numbers." |
| No data source | "How will you get NGX data?" | "Not solved yet. The official site blocks bots. The candidate has no terms page. It needs a decision, or a narrower scope." |
| One request at a time | "vLLM batches. Why don't you?" | "True. The client is synchronous for now. Batching or async is the first performance change." |
| README mentions missing files | "Where's ARCHITECTURE.md?" | "Not written yet. The README says 'once it exists'." |
| Small model | "A 7B model will be bad at this." | "Maybe. Measuring how much tools and data help a small model is part of the point." |

**Rule:** if asked for accuracy, latency or cost numbers, say "not measured in the repo yet" and describe how they'll be measured (the call log plus the graders).
