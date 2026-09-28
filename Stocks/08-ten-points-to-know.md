# Stocks: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Stocks

1. **Goal: answer factual questions on US and NGX stocks, and measure how often the answers are right.**
   *Why it matters:* the eval is the product, and the agent is what gets measured.

2. **Status: early. Built = config, model client, tests, CI. Not built = eval, agent, data, GPU runner.**
   *Why it matters:* never claim results that don't exist.

3. **US vs Nigerian stocks is deliberate: it tests memory vs reasoning.**
   *Why it matters:* it's the project's most interesting question.

4. **One YAML file per run, with Pydantic `extra="forbid"` and a 12-character config fingerprint.**
   *Why it matters:* typos fail loudly, and every result can be traced to its exact config.

5. **The client stack order is log → cache → retry → HTTP.**
   *Why it matters:* one cache entry per success, and cache hits are still counted honestly.

6. **The cache key is a SHA-256 of the entire request (`dataclasses.asdict`). Files are written atomically.**
   *Why it matters:* new settings can't silently reuse stale answers, and Kaggle kills can't corrupt the cache.

7. **Transient (timeout, 429, 5xx) vs permanent (other 4xx, bad body) errors. Backoff with jitter, max 4 retries, 30 s cap.**
   *Why it matters:* it protects a GPU budget of about 30 hours a week.

8. **64 tests. CI runs them twice, the second time with sockets disabled. `NeverCalledClient` proves the model wasn't touched.**
   *Why it matters:* tests can't secretly depend on the network or a GPU.

9. **Planned model: Qwen2.5-7B-Instruct-AWQ on vLLM, on a free Kaggle T4 (16 GB), temperature 0.**
   *Why it matters:* it explains the cost, memory and reproducibility choices.

10. **Data is the blocker: the NGX site sits behind a Sucuri bot filter, and the project refuses to bypass it.**
    *Why it matters:* it shows judgement on data ethics and terms of use.
