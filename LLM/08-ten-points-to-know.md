# LLM (gptlab): 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/LLM

1. **Trains GPT-style models of ~1M–100M parameters from scratch in plain PyTorch, on Kaggle 2× T4.**
   *Why it matters:* it shows you understand the full training stack, not just an API.

2. **Status: all code built, 170 tests pass on CPU, no GPU runs yet, no results.**
   *Why it matters:* never quote a loss or benchmark score.

3. **Data: FineWeb-Edu sample-10BT, light cleaning, exact dedup, 0.5% val split by text hash, ~2.6B train tokens planned.**
   *Why it matters:* clean splits prevent leakage, and the token budget suits a ~100M model.

4. **Tokenizer: byte-level BPE with 16,384 tokens, trained on train docs only. Stored as uint16 shards with a checked header.**
   *Why it matters:* a small vocab suits small models, and a checked header catches broken files.

5. **Model switches: learned/RoPE, LayerNorm/RMSNorm, pre/post norm, GELU/SwiGLU, weight tying, QK-norm, optional z-loss.**
   *Why it matters:* each switch is a planned ablation.

6. **Training: AdamW (decay on 2D only), warmup + cosine, fp16 + GradScaler, gradient accumulation, DDP with `no_sync`.**
   *Why it matters:* T4s have no bf16, so loss scaling is essential.

7. **Exact resume: weights, optimiser, scaler, step and all RNG states. Data position = step number. Tested to bit-identical losses (on CPU).**
   *Why it matters:* it's the core of surviving 11-hour sessions.

8. **Atomic checkpoints (`.tmp` → rename → `LATEST`), Hub keeps only `latest/` + `final/`, and history is squashed.**
   *Why it matters:* crash-safe, and inside free storage limits.

9. **Evaluation: val loss, perplexity, bits per byte (tokenizer-independent), zero-shot HellaSwag (acc and acc_norm, chance 25%), samples.**
   *Why it matters:* know what each metric means and when it's comparable.

10. **MFU = (6N + 12·L·T·d) × tokens/s ÷ GPU peak, checked against PyTorch's FLOP counter.**
    *Why it matters:* a common interview question on training efficiency.
