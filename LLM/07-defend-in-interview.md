# LLM (gptlab): Defending It in an Interview

Repo: https://github.com/MelvTheGoat/LLM

---

## 60-second pitch

> "I wrote a from-scratch PyTorch system to train small GPT models, from 1 to 100 million parameters, on free Kaggle GPUs. It covers the whole pipeline: cleaning FineWeb-Edu, a 16k byte-level BPE tokenizer, token shards, a transformer where RoPE, RMSNorm, pre/post norm, SwiGLU and QK-norm are all switches, and a training loop with fp16, a loss scaler, gradient accumulation and two-GPU DDP.
>
> The hard part is that Kaggle sessions end after 11 hours. So training stops itself early, saves atomically, including every process's random state, and the next session resumes exactly. There's a test showing that stop-and-resume gives bit-identical losses to training straight through. A job queue on a git branch lets many sessions work through experiments.
>
> 170 tests pass on CPU. The GPU runs, meaning scaling laws, ablations and HellaSwag, haven't happened yet, so I don't have result numbers to quote."

---

## Questions and honest answers

### 1. "Why write it from scratch instead of using Hugging Face?"
To understand and control every piece: the loss, the init, the LR schedule, the checkpoint format. Ablations need that control. The cost is more code to test, which is why there are 170 tests.

### 2. "Why a 16k vocabulary?"
My models are 128 to 768 wide. With 32k tokens, the embedding and output layers would hold most of a tiny model's parameters. The data job also measures bytes per token for 8k, 16k and 32k, so I can show the trade-off with numbers once it runs.

### 3. "Why fp16, not bf16?"
T4 GPUs don't support bf16. fp16 has a small range, so tiny gradients underflow to zero. The GradScaler multiplies the loss before backward and divides the gradients after, and it skips any step that overflows.

### 4. "How does exact resume work?"
The checkpoint holds the weights, AdamW state, scaler state, step number, and the RNG states of every process: Python, numpy, torch CPU and CUDA. The LR schedule is a function of the step. The data position is also just the step, because the loader reads a fixed seeded permutation. A test proves N straight steps equal N/2 + resume + N/2, exactly, with dropout on.

### 5. "Would that hold on a GPU?"
Not guaranteed. The test runs on CPU. On GPU, some kernels aren't deterministic, and `torch.compile` can change the order of operations. I'd expect very close but not always bit-identical. I haven't tested that on GPU yet.

### 6. "How do you know the model is correct?"
Tests: shapes, parameter counts, causal masking, tokenizer round-trip, the FLOP formula against PyTorch's counter, and gradient accumulation matching a bigger batch. Whether it *learns well* isn't measured yet. The smoke job is the first GPU step.

### 7. "What's MFU and how do you compute it?"
Model FLOPs utilisation: the FLOPs your model math needs per second, divided by the GPU's peak. I use 6N per token for the weights (2N forward, 4N backward) plus `12·layers·context·width` for attention's Q·Kᵀ and scores·V. For small models with long context, the attention term can be bigger than 6N.

### 8. "Why split train/val by hash?"
So a document's split doesn't depend on read order, and exact duplicates can't end up in both splits. That would leak training text into validation and make the loss look better than it is.

### 9. "Why store results on a git branch?"
It's free, versioned and readable anywhere. Each session writes only its own job's folder, and pushes with fetch, reset, copy, commit and push, retrying if someone else pushed first. At bigger scale I'd use a real tracker.

### 10. "How would you scale to 1B parameters?"
bf16 GPUs (A100/H100), FSDP or ZeRO to shard optimiser state, flash attention and sequence packing, ~20B tokens with MinHash near-dedup, object storage for checkpoints, and a proper scheduler and experiment tracker.

### 11. "What would break first?"
Probably fp16 stability on the larger models or at high learning rates. That's why there's optional QK-norm, z-loss and per-layer stats like max attention logit, and why experiment C is about instability. After that, Kaggle quotas and Hub upload times.

### 12. "Why FineWeb-Edu?"
It's openly licensed, already language- and near-duplicate-filtered, and its authors report better benchmark results per token than unfiltered web text. That matters when compute is tight.

### 13. "What's z-loss?"
An extra loss term: weight × mean of logsumexp² over the output logits. It keeps the softmax normaliser near zero so logits don't drift to huge values. It's from PaLM, and optional here.

### 14. "What do you expect the scaling experiment to show?"
Loss falling as a power law in parameters and compute, which I'd compare with the Chinchilla fit. I'd be careful: at 1M–100M parameters with limited seeds, the fit will have wide error bars. I won't claim a result until the runs exist.

---

## Weak spots and how to answer

| Weak spot | Likely poke | Honest answer |
|---|---|---|
| No GPU results | "So does it actually train?" | "It trains on CPU in the tests. The GPU smoke run is first in the queue. No real results yet." |
| Resume tested on CPU only | "GPU kernels aren't deterministic." | "Right. Bit-exactness is proven on CPU. On GPU I'd expect tiny differences and would measure them." |
| Tiny models on HellaSwag | "Won't that be ~25%?" | "Close to it for the smallest ones. It's useful as a trend across sizes, not an absolute score." |
| Git as results DB | "That won't scale." | "Agreed. It's fine for a few sessions. I'd move to a tracker at scale." |
| Missing REPORT/EXPERIMENTS docs | "README mentions files that don't exist." | "They're planned after the smoke test, and the README says so." |

**Rule:** for any loss, perplexity, HellaSwag or MFU number, say "not measured yet. The pipeline to measure it is built and tested."
