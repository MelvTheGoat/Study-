# Stocks: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Stocks

Use this on a whiteboard or in an interview. Be clear about what's **built** (config + model client + CI) and what's **planned** (agent, eval, data, GPU runner).

---

## Step 1: Requirements (2–3 min)

Say the problem in one line:
> "An agent that answers factual questions about US and Nigerian stocks, and an eval that measures how often it's right. The eval comes first."

**Functional**
1. Answer questions like "What was X's closing price on date D?" or "What's X's market cap?"
2. Cite where each number came from.
3. Never give buy or sell advice.
4. Score answers against known correct answers, the same way every time.
5. Compare three setups: closed book (model only), retrieval, and a full agent with tools.

**Non-functional**
- **Reproducible:** one YAML file defines a run, and results carry its fingerprint.
- **Deterministic:** temperature 0, fixed seed.
- **Cheap:** free GPU (Kaggle T4), about 30 GPU hours a week.
- **Honest:** no number shown until a real run produced it.
- **Legal data only:** no paid data, and no source whose terms forbid scraping.

---

## Step 2: Numbers and scale (1–2 min)

| Thing | Number | Source |
|---|---|---|
| GPU | 1 × T4, 16 GB | config comments |
| Model | 7B parameters, 4-bit AWQ, ~5 GB | `configs/smoke.yaml` |
| Context | 4,096 tokens | `max_model_len` |
| GPU budget | ~30 hours/week | `llm/cache.py` comment |
| Retries | up to 4, backoff 1 s → 30 s cap | `RetryingClient` |
| Smoke run | 5 questions | `limit: 5` |
| Eval size | **not built yet** | n/a |

**Say:** "Scale is tiny. The constraint is GPU time, not traffic, so caching and correct retries matter most."

---

## Step 3: High-level boxes (3 min)

```
[Run config YAML] -> [Agent loop] -> [Client stack: log -> cache -> retry -> HTTP] -> [vLLM on T4]
                          |
                          +-> [Tools: price lookup, calculator] -> [Data: NGX, US, SEC]
[Eval set (frozen as-of date)] -> [Agent] -> [Graders] -> [Results + call log]
```

Mark which boxes exist: the config and the client stack are built. The rest is planned.

---

## Step 4: Deep dive on each part (8–10 min)

### 4a. Run config
- Pydantic with `extra="forbid"`, because a typo must fail rather than be ignored.
- `fingerprint()` = first 12 characters of the SHA-256 of the sorted JSON dump.
- Sections: `model`, `eval`, `agent`, `seed`.

### 4b. The client stack (the part that's built)
Draw the layers and explain the order:
1. **Logging (outermost):** every call, including cache hits, becomes one JSONL line.
2. **Cache:** key = SHA-256 of the whole request. One file per key, sharded by the first 2 hex characters. Atomic write-then-rename.
3. **Retry:** transient errors only (timeout, connection error, 429, 5xx). Exponential backoff with jitter.
4. **HTTP:** OpenAI-compatible `/chat/completions`.

Why this order: retries inside the cache means one cache entry per success. Logging outside the cache means re-runs are counted honestly.

### 4c. Testing without a GPU
- `FakeModelClient`: scripted replies, matched by substring or queued.
- `NeverCalledClient`: proves a path skips the model.
- CI runs the tests twice, the second time with sockets disabled.

### 4d. The eval (planned; describe the design)
- Splits: `dev` (tune), `test` (only at the end), `hard` (hand-checked).
- A frozen `as_of` date, so "this year's return" has one answer forever.
- Graders: `numeric` (within tolerance), `exact`, `source` (the right citation).

### 4e. The agent (planned)
- `closed_book` and `retrieval` are baselines, `agent` has tools and `max_steps`.
- An optional `self_check` pass, to be measured to see if it pays for itself.

### 4f. Data (planned; the real blocker)
- NGX site: behind a Sucuri bot filter, so not used.
- african-markets.com: robots.txt allows it, but there's no terms page, so it needs a human decision.
- US: SEC EDGAR needs a contact email in the User-Agent header.

---

## Step 5: Bottlenecks (2 min)

1. **Data access:** NGX data is the hardest part, and it's not solved yet.
2. **GPU time:** Kaggle session limits. Handled by the cache and atomic writes.
3. **vLLM start-up:** connection refused while weights load. Handled by `is_ready()` plus transient retries.
4. **Eval leakage:** tuning on the test split. Handled by split discipline (planned).

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Eval first | Agent first | Stops fake progress | Nothing to demo yet |
| Small open model on free GPU | Hosted frontier API | Free, reproducible, own weights | Weaker model, session limits |
| File cache | Redis/DB | Zero setup, survives on disk | Slow at huge sizes |
| Strict config | Loose dicts | Typos can't mislabel results | More boilerplate |
| Refuse to bypass the bot filter | Scrape anyway | Legal and ethical | No NGX data yet |

Close with: "Next is the eval set and the NGX data source, then baselines, then the agent."
