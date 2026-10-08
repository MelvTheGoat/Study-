# Training a GPT from scratch on free GPUs that switch off after 11 hours

Repo: https://github.com/MelvTheGoat/LLM

## Why I built it

I use large language models every day, and I can explain the ideas behind them: attention, next-token prediction, scaling. But explaining an idea isn't the same as making it work. I wanted to train a GPT-style model myself, from the tokenizer to the evaluation, and run the kind of experiments that real labs run, just smaller.

So I built **gptlab**: a from-scratch PyTorch project that trains decoder-only transformers from about 1 million to 100 million parameters. The planned experiments are:

- **A. Scaling:** 5–6 model sizes, fit a power law, compare with Chinchilla.
- **B. Ablations:** RoPE vs learned positions, RMSNorm vs LayerNorm, pre- vs post-norm, SwiGLU vs GELU, warmup. Two seeds each.
- **C. Stability:** push the learning rate too high, diagnose the blow-up, fix it.
- **D. Efficiency:** fp16, `torch.compile`, 1 vs 2 GPUs, batch size.
- **E. Evaluation:** loss, perplexity, bits per byte, HellaSwag, samples.

To be clear up front: **none of these experiments have run yet.** The code is written and tested, and the first GPU job, a smoke test, passed on Kaggle's 2× T4 on 7 October. This post is about how the system is built and why.

## The problem

The constraint is money: I want this to cost nothing. That means:

- **Kaggle** for GPUs: two NVIDIA T4s, 16 GB each, **no bfloat16**, and a session that ends after about 11 hours.
- **Hugging Face Hub** for storing data and checkpoints.
- **A git branch** for logs and results.

So the real problem isn't "write a transformer". It's "train for days on machines that disappear every 11 hours, without losing a single step or getting a different answer than if they hadn't".

## How it works

### Data: FineWeb-Edu, cleaned lightly

I use FineWeb-Edu (`sample-10BT`): web pages filtered by a classifier for educational value, with an open licence. Its authors report better knowledge and reasoning scores than unfiltered web text for the same number of tokens. That matters when compute is tight.

It's already language-filtered and near-deduplicated within each crawl, so my cleaning is light: Unicode normalisation, dropping pages that are too short, too long, mostly symbols, or broken, and removing exact duplicates by hash. Every dropped document is counted with its reason in a `manifest.json`.

Two details matter more than they look:

1. **The split is decided by a hash of the text**, not by position. A document always lands in the same split whatever order I read it in, and an exact duplicate can never be in both train and validation.
2. **The tokenizer is trained on training documents only**, and the training loader refuses validation shards.

### Tokenizer: why 16k and not 32k

It's a byte-level BPE with 16,384 tokens. With model widths from 128 to 768, a 32k vocabulary would put most of the smallest models' parameters into the embedding and output layers. The data job also trains 8k, 16k and 32k tokenizers and measures bytes per token, so the trade-off will be shown with real numbers.

Tokens are stored as `uint16` (a 16k vocab fits in 2 bytes) in shards with a 1,024-byte header: a magic number, a version and a count. A truncated file fails on open instead of feeding garbage into training.

### The model

A standard decoder-only transformer, but every part I want to test is a switch:

- position encoding: learned or RoPE
- norm: LayerNorm or RMSNorm, placed before (pre) or after (post) each sub-layer
- MLP: GELU or SwiGLU, with matched parameter counts
- weight tying and QK-norm

It uses GPT-2 style initialisation. The detail I like is the scaled residual projections:

```python
s = std / math.sqrt(2 * self.cfg.n_layer) if getattr(module, "residual_proj", False) else std
nn.init.normal_(module.weight, mean=0.0, std=s)
```

There are `2 × n_layer` projections adding into the residual stream, so shrinking each one keeps the stream's size roughly the same at any depth when training starts.

### The training loop

Each optimiser step:
1. Sets the learning rate (linear warmup, then cosine decay).
2. Runs several micro-batches under fp16 autocast, adding up gradients (gradient accumulation). With two GPUs, gradients are only synced on the last micro-batch.
3. Unscales, clips and takes an AdamW step. Weight decay applies only to matrices, not norm gains or biases.
4. Logs loss, LR, gradient norm, tokens/s, MFU and memory. Every so often it also logs per-layer stats like attention entropy and the largest attention logit, which are early warning signs for instability.

Because T4s have no bfloat16, it's fp16 with a **loss scaler**. fp16 can't represent numbers smaller than about 6×10⁻⁸, so small gradients would round to zero. The scaler multiplies the loss by a big factor before backward, then divides the gradients back before the update. If anything overflows, it skips that step and lowers the factor.

## The hard parts, and how I solved them

### 1. Resuming exactly, not approximately

When the session ends, training must pick up as if nothing happened. A checkpoint saves:

- the model weights and the AdamW state
- the fp16 loss scaler state
- the step number (which is also the data position, see below)
- **the random-number state of every process** (Python, numpy, torch CPU and CUDA), because dropout draws from them

And there's a test that proves it:

```python
def test_resume_is_exact(tiny_data, tmp_path):
    """N steps straight == N/2 steps, stop, resume in fresh objects, N/2 more.

    Dropout is on, so this also checks that random states are restored.
    """
    ...
    a, b = losses(tmp_path / "straight"), losses(tmp_path / "resumed")
    assert a == b  # exact equality, every step
```

Exact equality, every step, plus identical final weights. A second test refuses to resume if the training config changed, while still letting logging settings change.

### 2. The data position is just a number

Most loaders keep state such as "which file, which offset, which shuffle buffer". Saving and restoring that is fiddly. My loader cuts all shards into blocks of `seq_len + 1` tokens and reads them in one seeded random permutation. Step `s` reads sequences `s·S` to `s·S + S − 1`, where `S` is the number of sequences per step.

So the position is the step number, and nothing else. A nice side effect: the same step reads the same data on one GPU or two, as long as the global batch size matches.

### 3. Saving without ever corrupting the last good checkpoint

A session can die in the middle of a save. So checkpoints are written into `step_X.tmp/`, renamed to `step_X/` only when complete, and only then is a `LATEST` file updated. A crash mid-save leaves the previous checkpoint untouched.

### 4. Free storage that doesn't fill up

Every version of a file on the Hugging Face Hub counts as storage. Overwriting a 1 GB checkpoint every 30 minutes would pile up fast. So the checkpoint repo keeps only `latest/` and `final/`, and after uploads I "super-squash" the repo history into one commit, which really deletes old versions.

### 5. Many sessions, one queue

A single notebook, pasted into Kaggle once, runs `python -m gptlab.runner`. It reads job statuses from a `results` git branch, claims the first job in `runs/queue.yaml` that can run on this hardware, and works on it. A background thread uploads checkpoints and pushes a heartbeat every few minutes. It stops 20 minutes before the 11-hour limit, saves, and marks the job "paused" for the next session.

Several sessions might push to the `results` branch at the same time, so every push starts from the newest remote state: fetch, reset, copy in only its own job's folder, commit, push, and repeat if someone got there first.

### 6. Measuring speed honestly

MFU (model FLOPs utilisation) tells you how much of the GPU you actually use. I count 6N FLOPs per token for the weights plus the attention term, `12 × layers × context × width`, which for small models with long context can be bigger than the 6N part. The formula is checked against PyTorch's own FLOP counter in the tests.

## What I learned

- **Fault tolerance is the real project** on free hardware. The transformer is maybe 270 lines. The resume, storage and queue logic is much more.
- **Make state tiny.** Turning the data position into the step number removed a whole class of resume bugs.
- **Test the property you care about directly.** "Resumed losses equal straight losses, exactly" is one test that covers a dozen ways to get resume wrong.
- **Pick the vocabulary for the model size.** A 32k vocab on a 128-wide model is mostly embedding.
- **fp16 needs care.** The loss scaler, QK-norm, z-loss and per-layer stats are all there to catch trouble early.

## What's next

1. ~~Run the smoke job on 2× T4~~ **Done (7 October).** The first attempt hung for 12 hours in the data step (forked workers deadlocked), costing about 12 GPU hours. The fix: `spawn` workers that fail loudly, and a runner that stops jobs gone silent for 30 minutes. The second attempt passed every step in 12.5 minutes. It also found two bugs: the original HellaSwag file now returns 404 (it now comes from a fixed Hugging Face revision), and the data manifest miscounted characters.
2. Build the full FineWeb-Edu dataset on a free CPU session (~2.6B tokens).
3. ~~Write `EXPERIMENTS.md` with the compute budget~~ **Done.** About 46 GPU hours for experiments A–E, from measured speeds: compiled fp16 on 2× T4 runs from about 625k tokens/s (2.9M params) to about 37k tokens/s (97.5M), at 11–20% MFU.
4. Run experiments A–E and write `REPORT.md`, with every number linked to a run log. Stage 1, the efficiency benchmark and learning-rate sweeps, is queued.

Until then, experiment loss, perplexity and HellaSwag are all **not measured yet**.
