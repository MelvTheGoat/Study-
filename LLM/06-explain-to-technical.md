# LLM (gptlab): Explained to an Engineer

Repo: https://github.com/MelvTheGoat/LLM

---

## Summary

From-scratch PyTorch pipeline for training decoder-only transformers (~1M–100M params) on Kaggle 2× T4. It covers data prep (FineWeb-Edu → clean → hash split → 16k byte-level BPE → uint16 shards), a configurable GPT, a training loop with fp16 + GradScaler, DDP, gradient accumulation and exact resume, evaluation (val loss, perplexity, bits per byte, zero-shot HellaSwag), a throughput/MFU benchmark, and a job runner that survives session limits using the HF Hub and a git `results` branch. **193 tests pass on CPU. The GPU smoke test passed on 2× T4 (7 Oct 2026); experiments A–E not run yet.**

~4,400 lines of Python in `gptlab/`, 24 commits, all on 27 Sep 2026. The default branch is `dev`.

## Architecture

```
gptlab/
  config.py          strict dataclass configs from YAML, --set overrides
  data/sources.py    FineWeb-Edu parquet, seeded block shuffle (~1000 docs)
  data/clean.py      NFC, filters, exact dedup, hash-based val split
  data/tokenizer.py  byte-level BPE via `tokenizers`
  data/shards.py     uint16 shards, 1024-byte header, memmap
  data/prepare.py    the data job, tokenizer comparison, manifest
  data/loader.py     TrainStream (permutation over blocks), ValData
  model.py           GPT, RMSNorm, RoPE, Attention, MLP, Block, param_counts
  optim.py           AdamW groups, LR schedule
  train.py           training loop, DDP, fp16, deadline stop, logs
  checkpoint.py      atomic checkpoints, RNG state, LATEST pointer
  flops.py           6N + attention FLOPs, MFU, GPU peak table
  evaluate.py        val loss, ppl, bits/byte, samples
  hellaswag.py       zero-shot scoring, acc and acc_norm
  bench.py           tokens/s, MFU, memory sweep
  hub.py             HF Hub store (+ LocalStore), history squash
  runner/            queue, results branch, job runner, DDP check
kaggle/runner.ipynb  paste once, runs the queue
```

## Key decisions

| Decision | Detail | Why |
|---|---|---|
| Plain PyTorch | No Lightning or HF Trainer | Full control and understanding |
| fp16 + GradScaler | T4 has no bf16 | Avoid gradient underflow |
| 16k vocab | vs 32k | At d=128–768, 32k would dominate parameter count |
| Hash-based split | 0.5% val by text hash | Order-independent. Duplicates can't cross splits. |
| Tokenizer on train only | Loader refuses val shards | No leakage |
| Position = step | Permutation over `seq_len+1` blocks | Exact resume from one integer, and GPU-count-independent |
| Save all RNG states | Python, numpy, torch CPU/CUDA, per rank | Dropout reproducibility across resume |
| Training hash | SHA-256 of config minus notes, eval and safe fields | Resume refuses a changed training config |
| Atomic checkpoints | `.tmp` → rename → update `LATEST` | Crash-safe |
| Hub history squash | Keep `latest/` + `final/` | Stay inside free storage |
| Git results branch | fetch → reset → copy own folder → commit → push, retry | Safe concurrent writers |
| Deadline stop | 11 h session, stop 20 min early | Time to save and upload |

## Model details

- Decoder-only. Token embedding, plus learned positions if `pos_encoding=learned`.
- **Attention:** fused QKV linear, `F.scaled_dot_product_attention(is_causal=True)`. Optional **RoPE** (rotate pairs `(i, i+d/2)`, θ configurable) and **QK-norm**. A debug path builds the full T×T matrix to log max logit and attention entropy.
- **MLP:** GELU or **SwiGLU** (fused gate/up, `silu(gate) * up`), with hidden size set for matched parameter counts.
- **Norms:** LayerNorm or **RMSNorm** (computed in fp32). **Pre** or **post** placement. With post-norm there's no final norm.
- **Init:** N(0, init_std). Residual projections use `init_std / sqrt(2·n_layer)`. Tied `lm_head` is skipped.
- **Loss:** cross-entropy computed from logsumexp in fp32, plus an optional **z-loss** `w · mean(lse²)` (PaLM) to keep logits from drifting.
- **Param counts:** total, embedding, and non-embedding N (Kaplan convention).

## Training details

- AdamW. Decay on `dim ≥ 2` tensors only. `fused=True` on CUDA.
- LR: linear warmup `lr·(step+1)/warmup`, then cosine to `lr·min_lr_ratio` (or linear/constant).
- Gradient accumulation with DDP `no_sync` on all but the last micro-batch.
- Unscale → clip → step. The scaler skips steps with inf/NaN.
- Optional `torch.compile`.
- Logs (JSONL per step): loss, LR, grad norm, tokens/s, MFU, memory. Debug stats every `debug_interval`: per-layer attention/MLP/residual RMS, attention entropy, max logit, logit size.

## Evaluation

- **Val loss** (nats), **perplexity** = e^loss.
- **Bits per byte** = total loss in bits ÷ UTF-8 bytes. Comparable across tokenizers.
- **HellaSwag zero-shot:** sum of ending-token log-probs given the context. `acc` (raw) and `acc_norm` (per byte). Uses the lm-eval-harness prompt format and the 10,042-example validation set. Chance is 25%.
- **MFU:** `(6N + 12·L·T·d) × tokens/s ÷ peak`, counting the full T×T (PaLM convention). Verified against PyTorch's FLOP counter.

## How it's tested

- **193 tests pass on CPU** (I ran them), up from 170, including a CPU test of compiled DDP training with gradient accumulation and a resume.
- Notable tests:
  - `test_resume_is_exact`: N straight vs N/2 + resume + N/2 gives identical losses, val losses and final weights (dropout on).
  - `test_resume_refuses_a_changed_config`.
  - `test_gradient_accumulation_matches_a_bigger_micro_batch`.
  - FLOP formula vs PyTorch's counter.
  - Tokenizer round-trip, shard header checks, cleaning and split behaviour, queue state rules, results branch writer.
  - A CPU DDP check so the multi-process path can be tested without GPUs.
- **GPU smoke test (`results/smoke-2`, 7 Oct):** passed every step in 12.5 min. Measured compiled fp16 throughput on 2× T4 from ~625k tok/s (s1, 2.9M params, 11.4% MFU) to ~37k tok/s (s6, 97.5M, 20.1% MFU). Compile speedup 1.5–2.6×, fp16 2.7× fp32, 2 GPUs 1.74× one. Resume on GPU differs by ≤0.0121 loss over 200 steps vs ≤0.0137 between two identical runs, so it's within noise. Full numbers and the ~46 GPU-hour budget are in `EXPERIMENTS.md`.
- **Evaluation of trained models: not measured yet.** No experiment runs and no `REPORT.md` yet.

## Known weaknesses

- **No experiment has run yet**, so none of the experiments have results. Only the smoke test has run on GPU.
- **Exact resume is bit-identical on CPU only.** On GPU it isn't (non-deterministic kernels), but the measured resume difference sits within run-to-run noise.
- **The first smoke attempt hung for 12 hours** (forked data workers deadlocked), costing ~12 of 30 weekly GPU hours. Fixed with `spawn`, loud worker failures, and a runner that stops silent or over-time jobs.
- **Small models on HellaSwag** will sit close to 25%, so differences may be within noise without many seeds.
- **Exact dedup only** across crawls.
- **Git as a results store** is fine for a few sessions, but not for many concurrent writers.
- **Squashing Hub history** removes old checkpoints permanently.
- **`REPORT.md` doesn't exist yet**; it fills in as runs finish.
- **Single data mix and one tokenizer size** for training (the comparison is only bytes/token).
