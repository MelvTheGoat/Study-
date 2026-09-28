# Stocks: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Stocks

---

## 1. LLM evaluation (evals)

**Simple explanation:** a fixed set of questions with known answers, graded the same way every time, so you can compare models and changes fairly.

**In this project:** designed in the config (`dev`/`test`/`hard` splits, frozen `as_of` date, `numeric`/`exact`/`source` graders). Not built yet.

**Also be ready to explain:**
- **Train/dev/test discipline**, and why you only run the test split once.
- **Numeric tolerance grading** vs exact match.
- **Grounding/citation checks:** is the source right, not just the number?
- **LLM-as-judge**: when it helps and why it's risky.
- **Contamination:** the model may have seen the answers during training (the US vs NGX angle).
- **Baselines:** closed book vs retrieval vs agent.

---

## 2. Agents and tool use

**Simple explanation:** a loop where the model can call tools (like a price lookup or a calculator), read the results, and decide when it's done.

**In this project:** planned (`agent.kind = agent`, `max_steps`, `tools`, `self_check`).

**Also be ready to explain:**
- **ReAct-style loops** (reason → act → observe).
- **Function/tool calling formats.**
- **Stopping rules** and step limits.
- **Self-verification** passes, and how to measure whether they're worth the cost.
- **Why tools help** with facts the model can't know (prices after its training cut-off).

---

## 3. Reliable API clients (retries, backoff, idempotency)

**Simple explanation:** retry only failures that might succeed next time, wait longer each time, and add randomness so clients don't all retry together.

**In this project:** `RetryingClient` and the transient/permanent split in `OpenAICompatibleClient`.

**Also be ready to explain:**
- **HTTP status classes:** 4xx (client error) vs 5xx (server error), and 429 (rate limit).
- **Exponential backoff with jitter**, and the "thundering herd" problem.
- **Idempotency:** why retrying a read is safe but retrying a payment isn't.
- **Timeouts** vs connection errors.
- **Circuit breakers.**

---

## 4. Caching

**Simple explanation:** save answers so you don't pay for the same work twice.

**In this project:** a disk cache keyed by a full-request SHA-256, sharded folders, atomic writes, corrupt-file-as-miss.

**Also be ready to explain:**
- **Cache keys and invalidation:** why the key must include every setting that affects the answer.
- **Atomic writes** (write temp, then rename).
- **Content-addressed storage.**
- **Cache hit rate** and how it skews latency stats.

---

## 5. Reproducibility and experiment tracking

**Simple explanation:** anyone should be able to re-run an experiment and get the same result, and know exactly what settings produced it.

**In this project:** one YAML per run, strict validation, config fingerprint, temperature 0, fixed seed, JSONL call log.

**Also be ready to explain:**
- **Determinism limits on GPUs** (non-deterministic kernels, batch effects).
- **Experiment trackers** (MLflow, Weights & Biases) and what they add.
- **Config management** (Pydantic, Hydra).

---

## 6. Serving open models (vLLM, quantisation)

**Simple explanation:** run your own model on a GPU and expose it through a standard chat API. Shrink it (quantise) so it fits.

**In this project:** vLLM with an OpenAI-compatible endpoint, Qwen2.5-7B at 4-bit AWQ (~5 GB) on a 16 GB T4, fp16 because the T4 has no bf16.

**Also be ready to explain:**
- **KV cache:** what it is and why it limits context length and batch size.
- **Quantisation:** AWQ vs GPTQ vs 8-bit, and the accuracy trade-off.
- **Continuous batching** and PagedAttention (why vLLM is fast).
- **Latency vs throughput.**

---

## 7. Testing with fakes

**Simple explanation:** replace slow or costly parts (like a GPU model) with predictable stand-ins so tests are fast and reliable.

**In this project:** `FakeModelClient`, `NeverCalledClient`, httpx mock transport, injected `sleep` and RNG, and a no-network CI pass.

**Also be ready to explain:**
- **Fakes vs mocks vs stubs.**
- **Dependency injection** and protocols (`ModelClient` is a `typing.Protocol`).
- **Testing time-based code** without real waiting.

---

## 8. Data sourcing, ethics and terms of use

**Simple explanation:** check that you're allowed to collect data before you collect it.

**In this project:** `DATA_SOURCES.md`, with robots.txt checks, terms checks, the WAF finding, the paid-portal decision and SEC User-Agent rules.

**Also be ready to explain:**
- **robots.txt** (what it is and isn't: it's not a licence).
- **WAFs and bot challenges.**
- **First-hand vs second-hand data**, and labelling.
- **Licensing and redistribution limits.**

---

## 9. Financial data basics

**Simple explanation:** what the questions will be about.

**Also be ready to explain:** share price vs market cap, adjusted vs unadjusted close (splits and dividends), total return, the P/E ratio, and why an **as-of date** is needed for any "current" figure. Also what SEC EDGAR filings are (10-K, 10-Q).

---

## 10. CI/CD hygiene

**Simple explanation:** automatic checks on every push.

**In this project:** ruff lint, pytest, a no-network pytest pass, and an authorship check on files and commits.

**Also be ready to explain:** branch filters (`branches-ignore` for data branches), `fetch-depth: 0` for history checks, and pip caching in Actions.
