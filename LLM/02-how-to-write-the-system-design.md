# LLM (gptlab): How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/LLM

Use this on a whiteboard. It's a **training system** design, not a serving system, so focus on data, the training loop, fault tolerance and evaluation.

---

## Step 1: Requirements (2 min)

One line:
> "Train small GPT models from scratch on free, time-limited GPUs, and run fair experiments on them."

**Functional**
1. Build a clean, tokenized dataset from FineWeb-Edu.
2. Train decoder-only transformers from ~1M to ~100M parameters.
3. Swap architecture parts for ablations (RoPE, RMSNorm, pre/post norm, SwiGLU, QK-norm).
4. Evaluate: val loss, perplexity, bits per byte, zero-shot HellaSwag, samples.
5. Benchmark speed: tokens/s, MFU, memory.
6. Run many jobs across many Kaggle sessions.

**Non-functional**
- **Survive session limits** (~11 hours): no lost work.
- **Exact resume:** the same losses as an uninterrupted run.
- **Reproducible:** config + git commit saved with each run.
- **Free:** Kaggle GPUs, Hugging Face Hub storage, git for logs.

---

## Step 2: Numbers (2 min)

| Thing | Number | Source |
|---|---|---|
| GPUs | 2× T4, 16 GB, fp16 only | README |
| Session limit | 11 h, stop 20 min early | `runs/queue.yaml` |
| Model sizes | ~1M–100M params, width 128–768 | README |
| Vocab | 16,384 (uint16 fits ≤ 65,536) | data config |
| Train tokens | ~2.6B (~5.2 GB of shards) | data config |
| Val split | 0.5% of docs, max 20M tokens | data config |
| Shard size | 100M tokens | data config |
| Smoke run | 4 layers, 4 heads, d=256, seq 512, 200 steps, 65,536 tokens/step | smoke config |

**Say:** "Data is ~5 GB and models are small. The hard constraint is the session time limit, not size."

A quick check on "compute-optimal": ~20 tokens per parameter (Chinchilla) × 100M params ≈ 2B tokens, which matches the 2.6B budget.

---

## Step 3: High-level boxes (3 min)

```
[FineWeb-Edu] -> [Clean + dedup + split] -> [BPE tokenizer] -> [uint16 shards] -> [HF Hub]
                                                                                   |
[queue.yaml] -> [Runner: claim job] -> [Loader] -> [GPT] -> [Train loop] -> [Checkpoints -> Hub]
                         |                                        |
                         +------------- [results git branch: status + logs] <---- [Eval]
```

---

## Step 4: Deep dive (10–12 min)

### 4a. Data
- Shuffle **blocks** of ~1,000 docs, because the files hold long runs from one crawl.
- Clean lightly (the source is already filtered), and count every drop with its reason.
- **Split by hash of text**, so a doc always lands in the same split and duplicates can't leak.
- Tokenizer trained on **train docs only**. The loader refuses val shards.
- uint16 shards with a checked header, read with memory-mapping.

### 4b. Loader (key idea for resume)
- One seeded permutation over all blocks.
- Position = a "global sequence number". Step `s` reads sequences `s·S … s·S+S−1`.
- **Resume needs only the step number.** 1 GPU and 2 GPUs read the same data per step.

### 4c. Model
Draw one block:
```
x -> Norm -> Attention(causal, fused SDPA, optional RoPE / QK-norm) -> + x
  -> Norm -> MLP(GELU or SwiGLU) -> + x
```
- GPT-2 init. The residual projections use `std / sqrt(2·n_layer)`.
- Optional weight tying, and optional z-loss on the logits.

### 4d. Training loop
1. LR from warmup + cosine.
2. Micro-batches with gradient accumulation. DDP `no_sync` except on the last.
3. fp16 autocast plus GradScaler (skip the step on overflow).
4. Unscale → clip → AdamW (decay on 2D tensors only).
5. Log loss, LR, grad norm, tokens/s, MFU, memory. Debug stats every N steps.

### 4e. Fault tolerance (the interesting part)
- **Atomic checkpoints:** write `step_X.tmp/`, rename, update `LATEST`.
- Save weights, AdamW state, scaler, step and **every rank's RNG state**.
- **Deadline stop** before the session limit, then "paused" status.
- **Background uploader** pushes checkpoints and a heartbeat.
- Test: N steps straight gives the same losses bit for bit as N/2 + resume + N/2.

### 4f. Job system
- `queue.yaml` lists jobs. Statuses live on the `results` branch.
- Claim → run → heartbeat → push. Several sessions push with a fetch-reset-retry loop.
- States: running, paused, done, diverged, failed (retry up to `max_attempts`).

### 4g. Evaluation
- Val loss (nats), perplexity = e^loss, bits per byte (tokenizer-independent).
- HellaSwag zero-shot: sum of log-probs per ending. `acc_norm` divides by bytes. Chance = 25%.
- MFU: (6N + attention term) × tokens/s ÷ GPU peak.

---

## Step 5: Bottlenecks (2 min)

1. **Session time limit:** handled by deadline stop, atomic checkpoints and exact resume.
2. **fp16 instability:** handled by the GradScaler, optional QK-norm, z-loss and debug stats (max attention logit, logit size).
3. **Hub storage growth:** handled by keeping only `latest/` and `final/`, plus squashing history.
4. **Concurrent result pushes:** handled by the fetch-reset-copy-commit-push retry.
5. **Memory:** the benchmark auto-searches the largest micro-batch that fits.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| From-scratch PyTorch | HF Trainer / nanoGPT copy | Learn and control every piece | More code to test |
| 16k vocab | 32k | Small models would waste parameters on embeddings | Longer token sequences |
| fp16 + scaler | bf16 | T4 has no bf16 | Overflow handling needed |
| git branch for results | Tracker service | Free, versioned, simple | Push conflicts, not queryable |
| Squash Hub history | Keep all checkpoints | Storage limits | No rollback |
| Exact dedup | MinHash near-dedup | Cheap, and the source already near-deduped per crawl | Cross-crawl near-dups remain |

Finish with: "No GPU runs yet. The smoke job is first in the queue, then the data job, then experiments A–E."
