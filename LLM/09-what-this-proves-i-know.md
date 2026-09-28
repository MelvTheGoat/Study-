# LLM (gptlab): What This Proves I Know

Repo: https://github.com/MelvTheGoat/LLM

---

## 1. The transformer architecture

**Simple explanation:** a stack of blocks, each with self-attention (tokens look at earlier tokens) and an MLP, joined by residual connections and normalisation.

**In this project:** `model.py`: causal attention, MLP, pre/post norm, residuals, GPT-2 init.

**Also be ready to explain:**
- **Q, K, V** and scaled dot-product attention: `softmax(QKᵀ/√d)·V`.
- **Causal masking.**
- **Multi-head attention**, and why heads split the width.
- **Residual stream** and why residual projections get smaller init.
- **Pre-norm vs post-norm** (training stability).
- **KV cache** at inference time (not built here, but expected knowledge).

---

## 2. Positional encoding (learned vs RoPE)

**Simple explanation:** the model needs to know word order. Learned positions add a vector per position. RoPE rotates queries and keys by an angle that grows with position, so attention depends on distance.

**Also be ready to explain:** why RoPE extrapolates better, what θ (base frequency) does, and ALiBi as an alternative.

---

## 3. Normalisation and activation choices

**Simple explanation:** LayerNorm centres and scales. RMSNorm only scales (cheaper). SwiGLU is a gated MLP used by LLaMA.

**Also be ready to explain:** why SwiGLU uses a 2/3 hidden size to match parameters, why norms are computed in fp32 under fp16, and QK-norm for attention logit growth.

---

## 4. Tokenization (BPE)

**Simple explanation:** start from 256 bytes and repeatedly merge the most frequent pair into a new token until you hit the vocab size.

**In this project:** byte-level BPE, 16,384 tokens, trained on train docs only, with an 8k/16k/32k comparison on bytes per token.

**Also be ready to explain:** byte-level vs character-level, vocab size trade-offs (sequence length vs embedding size), and why bits per byte beats per-token loss across tokenizers.

---

## 5. Training dynamics and optimisation

**Simple explanation:** AdamW adapts each weight's step size. Warmup avoids early blow-ups. Cosine decay slows down near the end.

**Also be ready to explain:**
- **Adam's moments** (β1, β2) and **decoupled weight decay** (AdamW vs Adam + L2).
- Why **no decay on norms and biases**.
- **Gradient clipping.**
- **Gradient accumulation** = a bigger effective batch.
- **Loss spikes and divergence:** causes (LR too high, logit growth) and fixes (lower LR, QK-norm, z-loss, warmup).

---

## 6. Mixed precision

**Simple explanation:** compute in 16-bit to go faster and use less memory, keeping sensitive parts in 32-bit.

**In this project:** fp16 autocast plus GradScaler on T4s.

**Also be ready to explain:** fp16 vs bf16 range and precision, loss scaling, master weights, and when a step gets skipped.

---

## 7. Distributed training

**Simple explanation:** copy the model on each GPU, give each GPU different data, and average gradients.

**In this project:** PyTorch DDP over 2 GPUs, `no_sync` during accumulation, rank-aware eval and checkpoints.

**Also be ready to explain:** all-reduce, DDP vs FSDP/ZeRO vs tensor/pipeline parallelism, and `torchrun`.

---

## 8. Scaling laws

**Simple explanation:** loss falls predictably (a power law) as you add parameters, data and compute.

**In this project:** planned experiment A (5–6 sizes, power-law fit, Chinchilla comparison). Uses Kaplan's non-embedding N.

**Also be ready to explain:** Kaplan (2020) vs Chinchilla (2022), ~20 tokens per parameter as compute-optimal, and C ≈ 6·N·D.

---

## 9. Evaluation of language models

**Simple explanation:** measure how well the model predicts held-out text, and test it on standard tasks.

**In this project:** val loss, perplexity, bits per byte, zero-shot HellaSwag (acc and acc_norm), samples.

**Also be ready to explain:** perplexity = e^loss, why acc_norm normalises by length, contamination, few-shot vs zero-shot, and lm-evaluation-harness.

---

## 10. Efficiency measurement

**Simple explanation:** how much of the GPU's power you actually use.

**In this project:** MFU with 6N + attention FLOPs, tokens/s, memory, and an auto micro-batch search.

**Also be ready to explain:** MFU vs HFU, memory-bound vs compute-bound, flash attention, `torch.compile`, and activation checkpointing.

---

## 11. Data engineering for pretraining

**Simple explanation:** clean, deduplicate, split and pack text so the model learns from good, non-repeated data.

**In this project:** block shuffle, NFC normalisation, filters, exact dedup, hash split, uint16 memmapped shards, and a manifest with drop reasons.

**Also be ready to explain:** MinHash near-dedup, quality classifiers (how FineWeb-Edu was filtered), data mixtures, and sequence packing.

---

## 12. Fault-tolerant systems

**Simple explanation:** design so a crash or shutdown loses nothing and can resume.

**In this project:** atomic checkpoints, RNG state, deadline stop, heartbeat, job states, and concurrent-safe git pushes.

**Also be ready to explain:** idempotency, atomic rename, optimistic concurrency (retry on conflict), and preemptible or spot instances.
