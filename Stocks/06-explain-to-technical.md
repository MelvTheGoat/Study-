# Stocks: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Stocks

---

## Summary

A planned LLM agent for factual questions on US and NGX stocks, with the eval harness built first. **Built today:** a strict run config, a layered model client (log → cache → retry → HTTP), fake clients, 64 tests, and CI with a no-network pass. **Not built:** eval set, graders, agent, data pipeline, Kaggle/vLLM runner. No results exist.

~1,360 lines of Python including tests. Runtime dependencies: pydantic, pyyaml, httpx.

## Architecture

```
src/stockagent/
  config.py            RunConfig = ModelConfig + EvalConfig + AgentConfig + seed
  llm/base.py          ModelClient protocol, ChatRequest/Response, error types
  llm/openai_client.py httpx client for /v1/chat/completions
  llm/retry.py         RetryingClient (transient only, backoff + jitter)
  llm/cache.py         ResponseCache (sharded JSON files) + CachingClient
  llm/calllog.py       CallLog (JSONL) + LoggingClient
  llm/fake.py          FakeModelClient, NeverCalledClient
  llm/__init__.py      build_client(), request_from_config()
  collectors/          empty (NGX collector planned)
configs/smoke.yaml     5-question smoke run, closed_book, Qwen2.5-7B-AWQ
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| Strict config | Pydantic `extra="forbid"` on every model | Typos fail loudly instead of mislabelling results |
| Config fingerprint | `sha256(sorted JSON)[:12]` | Quick "same run?" check |
| Greedy decoding by default | `temperature=0.0`, `top_p=1.0`, `seed=0` | An eval that moves when nothing changed can't detect small gains |
| Sampling only from config | `request_from_config()` | The config and the cache key always match what was asked |
| Full-request cache key | `sha256(json(asdict(request)))` | New fields join the key automatically |
| Sharded file cache | `key[:2]/key[2:].json`, write tmp then `os.replace` | Atomic under Kaggle kills. Fast listing. Easy to sync. |
| Corrupt cache entry = miss | JSON decode error → miss, then overwrite | Half-written files don't fail a run |
| Error taxonomy | Transient: timeout, transport error, 429, 5xx. Permanent: other 4xx, bad body. | Don't burn GPU time on hopeless retries |
| Backoff | `min(1·2^n, 30) × U(0.5, 1.0)`, max 4 retries | Jitter avoids retry storms |
| Stack order | log → cache → retry → HTTP | One cache write per success. Cache hits still logged. |
| Latency honesty | `cached` flag. Cache hits report `latency_ms=0`, and the original latency is kept in the file. | A median that mixes hits and misses is useless |
| Injectable transport, sleep, RNG | httpx `transport`, `sleep`, `rng` fields | Tests never open sockets or really wait |

## Models and algorithms

- **Planned model:** `Qwen/Qwen2.5-7B-Instruct-AWQ`, fp16 activations (a T4 has no bf16), `max_model_len` 4096, served by vLLM with an OpenAI-compatible API. `gpu_memory_utilization` defaults to 0.90.
- **Planned agent kinds:** `closed_book` (model only), `retrieval` (context stuffed in), `agent` (tool loop up to `max_steps`, optional `self_check`).
- **Planned graders:** `numeric`, `exact`, `source`.
- **Planned splits:** `dev`, `test`, `hard`. Each eval version freezes an `as_of` date.
- No retrieval, ranking or tool-use algorithm is implemented yet.

## How it's tested

- **64 tests pass** (~1.5 s). Areas: config validation, request and cache keys, cache/retry/log plumbing, and the OpenAI-compatible client against an httpx mock transport.
- **CI (`tests.yml`)**: ruff lint → pytest → pytest with `pytest-socket --disable-socket`.
- **CI (`authorship.yml`)**: fails if files or commit messages/authors contain AI-tool attribution strings.
- **Evaluation of the agent: not measured in the repo yet.** No eval set, no runs, no results branch content.

## Known weaknesses

- **Most of the system doesn't exist yet.** The agent, eval, graders, data and runner are all planned.
- **No data source is settled.** The NGX site is blocked by a Sucuri WAF. The alternative site has no terms page. US sources aren't checked yet.
- **Synchronous, one request at a time.** vLLM gets most of its throughput from batching, which this client doesn't use yet.
- **File-per-entry cache** won't scale to very large runs.
- **`CallLog.read()` loads the whole file** into memory. Fine now, but not at millions of lines.
- **README points at `ARCHITECTURE.md`**, which doesn't exist yet. It also mentions `runs/` and `data/samples/`, which aren't in the repo.
- **Seed support varies by server.** The client sends `seed`, but determinism also depends on vLLM's settings and hardware.
- **Earlier TypeScript prototype removed.** Its retrieval and guardrail tests don't carry over.
