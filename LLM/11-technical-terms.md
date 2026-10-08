# Training a Small LLM from Scratch: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow the path of the work: preparing the text, building the model, training it, measuring it, and running jobs on free computers.

---

## 1. Preparing the Text

### Dataset (FineWeb-Edu)
**What it means:** a dataset is a big collection of examples to learn from. FineWeb-Edu is a free collection of educational web pages.

**Why it's needed here:** a model learns only from the text it's shown. This one is already filtered for quality, so the project's own cleaning can stay light.

### Parquet
**What it means:** a file format for storing large tables of data compactly.

**Why it's needed here:** FineWeb-Edu is shared as parquet files, so the project has to read them.

### Deduplication (dedup)
**What it means:** removing copies of the same document.

**Why it's needed here:** repeated pages make the model memorise instead of learn. The project removes exact copies, but not near-copies.

### Train/validation split
**What it means:** keeping some text aside (validation) that the model never trains on, to test it fairly.

**Why it's needed here:** a model always looks good on text it has seen. Here, 0.5% of pages are set aside.

### Hash
**What it means:** a short fingerprint made from some data. The same data always gives the same fingerprint.

**Why it's needed here:** each page's fingerprint decides which group it goes in. So a page and its exact copy always land together, and can't leak from training into testing.

### Token and tokenizer
**What it means:** a token is a small piece of text. The tokenizer splits text into tokens and gives each a number.

**Why it's needed here:** models only understand numbers. The project's tokenizer learns 16,384 tokens from training text only.

### BPE (byte pair encoding)
**What it means:** a way to build a tokenizer by repeatedly joining the most common pairs of pieces into bigger pieces. "Byte-level" means it starts from raw bytes, so it can handle any text.

**Why it's needed here:** it's the standard method for GPT-style models. The project also compares list sizes of 8,000, 16,000 and 32,000 to see how much each squeezes the text.

### Vocabulary size
**What it means:** how many different tokens the tokenizer knows.

**Why it's needed here:** a bigger vocabulary needs more parameters to store. With 32,000, most of a tiny model would be spent on the word list, so 16,384 was chosen.

### Shard
**What it means:** one piece of a large dataset split into many files.

**Why it's needed here:** the 2.6 billion tokens are stored as 100-million-token files. Each file starts with a check label, so a broken file is spotted straight away.

### Memory-mapping
**What it means:** reading a file as if it were already in memory, loading only the parts you touch.

**Why it's needed here:** the token files are gigabytes big. Memory-mapping lets training jump to any spot without loading everything.

---

## 2. Building the Model

### Parameter
**What it means:** one adjustable number inside a model. Training tunes them.

**Why it's needed here:** model size is counted in parameters. This project trains models from about 1 million to 100 million.

### Transformer (decoder-only, GPT-style)
**What it means:** the design used by ChatGPT-style models. It reads the text so far and predicts the next token. "Decoder-only" means it only generates, and doesn't translate from one input to another.

**Why it's needed here:** it's the model being built and studied.

### Attention
**What it means:** the part of a transformer that lets each token look back at earlier tokens and decide which ones matter.

**Why it's needed here:** it's how the model uses context. The project uses PyTorch's fast built-in version.

### Position encoding (learned or RoPE)
**What it means:** a way to tell the model where each token sits in the sentence. "Learned" gives each position its own numbers. RoPE rotates the numbers by an amount based on position.

**Why it's needed here:** without it, "dog bites man" and "man bites dog" look the same. Both options are included so they can be compared.

### Normalisation (LayerNorm or RMSNorm)
**What it means:** rescaling numbers inside the model so they stay in a sensible range.

**Why it's needed here:** without it, numbers can grow or shrink until training breaks. Both types, and where they go ("pre" or "post"), can be switched for experiments.

### MLP (GELU or SwiGLU)
**What it means:** the "thinking" layer that processes each token after attention. GELU and SwiGLU are two versions of it.

**Why it's needed here:** SwiGLU is used in many newer models. Being able to switch lets the project test whether it helps.

### Ablation
**What it means:** an experiment where you change one part and see what difference it makes.

**Why it's needed here:** every switch in the model exists so it can be tested this way.

---

## 3. Training the Model

### Loss (cross-entropy)
**What it means:** a score for how surprised the model is by the real next token. Lower is better.

**Why it's needed here:** training works by pushing this number down.

### Optimiser (AdamW)
**What it means:** the method that decides how much to nudge each parameter after each step. AdamW is a popular one.

**Why it's needed here:** it's the standard, well-understood choice for training transformers.

### Learning rate, warmup and cosine decay
**What it means:** the learning rate is how big each nudge is. Warmup starts small and grows. Cosine decay then shrinks it smoothly towards the end.

**Why it's needed here:** big nudges at the start can break training, and small ones at the end help it settle.

### Gradient
**What it means:** for each parameter, which way to nudge it, and how much, to reduce the loss.

**Why it's needed here:** training is just following gradients. The project also caps very large gradients ("clipping") so one bad step can't ruin everything.

### Batch and gradient accumulation
**What it means:** a batch is a group of examples trained on together. Gradient accumulation adds up several small batches before taking one step.

**Why it's needed here:** a big batch may not fit in GPU memory. Adding up small ones gives the same result, and a test proves it.

### fp16 and loss scaling
**What it means:** fp16 is a shorter number format that's faster but can't hold very tiny numbers. Loss scaling makes numbers bigger during training so they don't round down to zero.

**Why it's needed here:** the free T4 GPUs don't support the safer short format (bf16), so fp16 plus a loss scaler is the only fast option.

### DDP (distributed data parallel)
**What it means:** running a copy of the model on each GPU, each working on different data, then combining what they learned.

**Why it's needed here:** Kaggle gives 2 GPUs, so this uses both. A test checks this on a normal computer too.

### Exact resume
**What it means:** stopping training and restarting it later, with exactly the same result as if it never stopped.

**Why it's needed here:** free sessions end after about 11 hours. A test proves that stopping and restarting gives identical results on a normal computer. On the real GPUs it's not bit-for-bit identical, but the GPU test run showed the difference is no bigger than normal run-to-run noise.

### RNG state
**What it means:** RNG means random number generator. Its state is where it is in its sequence of random numbers.

**Why it's needed here:** some training steps use randomness. Saving the state means a resumed job gets the same "random" numbers as before.

### Atomic checkpoint
**What it means:** saving a snapshot so it's either complete or not there at all, never half-written.

**Why it's needed here:** a crash while saving would otherwise destroy the last good save.

---

## 4. Measuring the Model

### Perplexity
**What it means:** another way of showing the loss: roughly, how many tokens the model is "choosing between" at each step. Lower is better.

**Why it's needed here:** it's a common, easy-to-compare number for language models.

### Bits per byte
**What it means:** how much information the model needs for each byte of text. Lower is better.

**Why it's needed here:** loss depends on the tokenizer, but bits per byte doesn't. So models with different tokenizers can be compared fairly.

### HellaSwag (zero-shot)
**What it means:** a multiple-choice test where the model picks the most sensible ending to a short story. "Zero-shot" means with no example answers given first.

**Why it's needed here:** it's a standard test for small models. Guessing gets 25%, so tiny models will score near that. It's useful for spotting a trend across sizes.

### FLOPs and MFU
**What it means:** FLOPs count the maths operations. MFU (model FLOPs utilisation) is how much of the GPU's top speed you actually use.

**Why it's needed here:** it shows how efficient training is. The project's formula is checked against PyTorch's own counter.

### Tokens per second (throughput)
**What it means:** how many tokens training gets through each second.

**Why it's needed here:** it's the main speed number for the planned efficiency experiments.

### Scaling law
**What it means:** a pattern showing how loss improves as models and data get bigger.

**Why it's needed here:** measuring it is one of the planned experiments. The data size (2.6 billion tokens) is set for a model of about 100 million parameters.

---

## 5. Running Jobs on Free Computers

### Kaggle session
**What it means:** a free period of computer time on Kaggle, with GPUs, that ends after a time limit.

**Why it's needed here:** it's the only hardware the project uses. Everything is designed around sessions ending.

### Deadline stop
**What it means:** stopping on purpose before time runs out.

**Why it's needed here:** training stops 20 minutes before the 11-hour limit, to leave time to save and upload.

### Job queue
**What it means:** a to-do list of jobs, each with a status like running, paused, done or failed.

**Why it's needed here:** one notebook works through all the experiments across many sessions, always picking the next job that can run.

### Heartbeat
**What it means:** a regular "I'm still alive" signal from a running job.

**Why it's needed here:** a running job sends one every 10 minutes, so other sessions leave it alone. If a job goes silent for 30 minutes, its session probably died, so the job becomes free for another session to pick up.

### Results branch
**What it means:** a git branch (a separate line of saved files) used only to store job statuses and logs.

**Why it's needed here:** it's a free, simple place to store results. If two sessions save at once, one simply tries again.

### History squashing
**What it means:** replacing a storage folder's full history of changes with a single snapshot.

**Why it's needed here:** free Hugging Face storage grows with every saved version. Squashing keeps it small, but old checkpoints are gone for good.
