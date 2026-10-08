# Training a Small LLM from Scratch: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: all of this is built and tested, and a first GPU test run passed on 7 October 2026. But the real training experiments haven't run yet, so there are no experiment results so far.*

---

## Okay, so what are we building?

Alright. Loads of people use ChatGPT. Very few can explain how one is actually trained. So here's our project: we build everything needed to train small ChatGPT-style models **from scratch**, ourselves.

"Small" means about 1 million to 100 million parameters. A parameter is one of the adjustable numbers inside a model. ChatGPT-sized models have billions, so ours are tiny, but they work the same way.

And we do it on **free computers**. Kaggle gives free sessions with 2 GPUs, special chips that do lots of maths at once.

The catch? Each session switches off after about 11 hours. So the real engineering challenge is this: how do you run long training jobs on computers that keep switching off, without losing any work?

## So what do we need?

Let's list it out:

1. **Settings** that describe every run exactly.
2. **Text** to learn from, lots of it, and clean.
3. **A test set** of text the model never trains on.
4. **A tokenizer**, which turns text into numbers.
5. **Compact storage** for billions of those numbers.
6. **A way to feed batches** to the model, that can restart exactly.
7. **The model itself.**
8. **A training loop** that uses both GPUs and stops in time.
9. **Safe saving**, so a crash never ruins progress.
10. **Ways to measure** how good the model is, and how fast training runs.
11. **A job system** that works through experiments across many short sessions.

That's the list. The order is: settings, then data, then model, then training, then measuring, then the job system that ties it all together on Kaggle.

## Step zero: set up the workshop

First, a `README.md`, the front page. It lists the planned experiments, and honestly marks each one "not run yet" until it has run. A `.gitignore` stops git saving big things like downloaded data and checkpoints.

`requirements.txt` lists the tools: PyTorch, NumPy, PyYAML, tokenizers, huggingface_hub, pyarrow, matplotlib and pytest. One nice note in it: PyTorch's version isn't pinned, so Kaggle keeps its own GPU-ready build. `pyproject.toml` describes the project as a package.

The code lives in a folder called `gptlab`, with `__init__.py` files marking each folder as a package. The top one says it plainly: "a small GPT language model written from scratch in PyTorch."

And `RUNNING.md` explains the one-time Kaggle setup: making a Hugging Face token, a GitHub token, and the notebook. After that, every run is one click.

> **📁 Files we just created**
> - `README.md`: the front page, marking each experiment "not run yet" until it runs.
> - `.gitignore`: files git should not save, like data and checkpoints.
> - `requirements.txt`: the tools to install.
> - `pyproject.toml`: the project's package details.
> - `gptlab/__init__.py`, plus `__init__.py` in `gptlab/data` and `gptlab/runner`: mark folders as packages.
> - `RUNNING.md`: how to set up Kaggle once.

## Step one: settings for every run

Before anything else, `gptlab/config.py`. Every run is described by one small YAML settings file. That file only lists what's different from the defaults.

Here's the clever part. When a run starts, the *full* settings, every single value including defaults, are saved next to its logs, along with the exact version of the code. So any result can be re-run exactly.

And there's a safety check. Resuming a run refuses if the training settings have changed since it started. You can't accidentally continue an experiment with different settings and mix up the results.

Tested by `tests/test_config.py`, and by `tests/test_configs_valid.py`, which loads every settings file in the project. As its note says, that "catches typos before Kaggle does".

> **📁 Files we just created**
> - `gptlab/config.py`: one settings file per run, with the full version saved.
> - `tests/test_config.py`: tests the settings.
> - `tests/test_configs_valid.py`: loads every settings file to catch typos.

## Step two: the text

Now the data. We use FineWeb-Edu, a free collection of educational web pages on Hugging Face (a website for sharing AI models and data). It comes as parquet files, a compact table format.

`gptlab/data/sources.py` reads them. And here's a subtle problem it solves. Inside each file, pages are grouped by when they were collected from the web. Read a file front to back, and your first data shard is all from one crawl.

So it reads pages in blocks of about 1,000, in a shuffled order. That way every shard gets a good mix.

Then `gptlab/data/clean.py` cleans them. FineWeb-Edu is already filtered for quality, so cleaning stays light. It tidies text, drops pages that are too short, too long, broken or mostly not letters, and removes exact copies. And it counts every dropped page with its reason, so nothing disappears silently.

## Setting aside the test text

`clean.py` also does something really important: it sets aside 0.5% of pages as test text, which the model never trains on.

But how does it choose? Not randomly. It uses a fingerprint of each page's text. Here's the real function:

```python
def is_val(digest: bytes, val_fraction: float) -> bool:
    """Deterministic split: about `val_fraction` of all documents go to validation."""
```

The same page always lands in the same group. So if a page appears twice, both copies go to the same side. A copy can never sneak from training into testing.

> **📁 Files we just created**
> - `gptlab/data/sources.py`: reads web pages in a shuffled order.
> - `gptlab/data/clean.py`: cleans, removes copies, and sets aside test text.

## Step three: text into numbers

Models only understand numbers. So `gptlab/data/tokenizer.py` builds a tokenizer. It uses BPE, byte pair encoding. It starts from single bytes and keeps joining the most common pairs into bigger pieces, until it has 16,384 pieces.

Why 16,384 and not more? Because a bigger list, like 32,000, would use up most of a tiny model's parameters just storing the word list. The project also compares 8,000, 16,000 and 32,000 to see how much each squeezes the text. And the tokenizer only ever learns from training text, never the test set.

Then we need to store billions of numbers compactly. `gptlab/data/shards.py` writes them into files called shards. Each number takes just 2 bytes. Each file starts with a small header containing a magic number, a version and a count:

```python
MAGIC = 0x67707431  # the bytes "gpt1"
```

So if a file is broken or the wrong type, it's caught the moment it's opened.

And `gptlab/data/prepare.py` runs the whole data job: download, clean, remove copies, train the tokenizer, turn text into numbers, write the shards, and upload them. Its settings are in `configs/data/fineweb_edu_16k.yaml`: 4 files, about 2.9 million pages, aiming for 2.6 billion training tokens. That's about 5.2 GB, in files of 100 million tokens each. There's also a tiny version, `configs/data/smoke.yaml`, for quick checks.

Tests: `tests/test_clean.py`, `tests/test_tokenizer.py`, `tests/test_shards.py` and `tests/test_prepare.py`. They use a small sample of text in `tests/data/sample_docs.jsonl`, which is lines from *Alice in Wonderland*.

> **📁 Files we just created**
> - `gptlab/data/tokenizer.py`: builds the 16,384-piece tokenizer.
> - `gptlab/data/shards.py`: stores token numbers compactly, with a checked header.
> - `gptlab/data/prepare.py`: runs the whole data job.
> - `configs/data/fineweb_edu_16k.yaml`: settings for the real dataset.
> - `configs/data/smoke.yaml`: settings for a tiny test dataset.
> - `tests/test_clean.py`, `tests/test_tokenizer.py`, `tests/test_shards.py`, `tests/test_prepare.py`: data tests.
> - `tests/data/sample_docs.jsonl`: a small sample of text for tests.

## Step four: feeding the model

Now we need to hand the model batches of text. That's `gptlab/data/loader.py`, and it holds one of the cleverest ideas in the project.

The text is cut into blocks. Every pass through the data uses one fixed, shuffled order of all the blocks, set by a seed number. So the position in the data is just one number: the step count.

Why does that matter so much? Because to restart a stopped job exactly, all we need is the step number. And there's a bonus: training on 1 GPU and training on 2 GPUs read exactly the same data at each step.

Tested in `tests/test_loader.py`.

Let's pause and look at where we are. We have exact settings. We have clean text, with a test set that can't leak.

We have a tokenizer, compact storage, and a loader that can restart from one number. The data side is done. Now the model.

> **📁 Files we just created**
> - `gptlab/data/loader.py`: feeds batches in a fixed order, so one number marks the position.
> - `tests/test_loader.py`: tests the loader.

## Step five: the model

`gptlab/model.py` is the model itself, a GPT-style transformer written from scratch. It reads the text so far and predicts the next token.

And here's the design choice: almost every part can be switched, so we can run fair experiments. How it tracks word positions, which kind of rescaling layer it uses and where it goes, and which kind of "thinking" layer it uses. Like a car where you can swap parts and test-drive each version.

It also uses PyTorch's fast built-in attention, which is the part that lets each word look back at earlier words.

`gptlab/generate.py` lets the model write text, one token at a time. That's how we'll see sample output.

Tested in `tests/test_model.py`.

> **📁 Files we just created**
> - `gptlab/model.py`: the GPT model, with swappable parts.
> - `gptlab/generate.py`: makes the model write text.
> - `tests/test_model.py`: tests the model.

## Step six: training it

Now the training loop. First, `gptlab/optim.py` sets up how the model learns. It uses AdamW, a popular method for nudging the parameters. The learning rate starts small, grows during a warm-up, then shrinks smoothly towards the end.

Then `gptlab/train.py`, the loop itself. Each step: feed in text, measure how wrong the guesses are (the "loss"), and nudge the parameters to do better.

There are a few special touches here. The free T4 GPUs don't support the safer short number format. So it uses an older one, plus a "loss scaler" that stops tiny numbers turning into zero.

It also adds up several small batches before taking one step, when a big batch won't fit in memory. A test proves that gives the same result as one big batch. And it uses both GPUs at once. That's helped by `gptlab/distributed.py`, which has small helpers for running one copy of the model per GPU.

Then the most important touch: **the deadline**. Here's the real check inside the loop:

```python
if deadline is not None and time.time() >= deadline:
    stop, reason = True, "deadline"
```

When time's up, it stops, saves and leaves. That's how we beat the 11-hour limit.

`gptlab/metrics.py` writes a log line for every step, like loss, speed and memory. It also records detailed stats layer by layer, which help spot training going wrong.

Tested in `tests/test_optim.py` and `tests/test_train.py`. The settings for a small test training run are in `configs/smoke/train.yaml`.

> **📁 Files we just created**
> - `gptlab/optim.py`: how the model learns, and the learning rate schedule.
> - `gptlab/train.py`: the training loop, with the deadline stop.
> - `gptlab/distributed.py`: helpers for using 2 GPUs at once.
> - `gptlab/metrics.py`: a log line per step, and detailed layer stats.
> - `configs/smoke/train.yaml`: settings for a small test training run.
> - `tests/test_optim.py` and `tests/test_train.py`: training tests.

## Step seven: saving safely

Now, what happens when the deadline hits, or something crashes? That's `gptlab/checkpoint.py`.

A checkpoint saves everything needed to continue as if nothing happened: the model, the learning state, the loss scaler, the data position, and even the random number settings for every GPU.

And it saves safely. Files go into a temporary folder first, which is renamed when complete. Only then is a small "latest" pointer updated. So a crash while saving can never ruin the last good checkpoint.

And here's the proof. `tests/test_checkpoint.py` trains straight through, then trains half, stops, restarts, and trains the other half. It checks the two give exactly identical results. That's been proven on a normal computer, but not yet on a GPU.

> **📁 Files we just created**
> - `gptlab/checkpoint.py`: saves everything safely, for an exact restart.
> - `tests/test_checkpoint.py`: proves stop-and-restart gives identical results.

## Step eight: measuring it

Now, how good is the model, and how fast is training?

`gptlab/evaluate.py` measures quality. It gives the loss on the test text, plus perplexity and bits per byte, which are other ways of showing it. Bits per byte is fair to compare across different tokenizers.

`gptlab/hellaswag.py` runs a standard quiz called HellaSwag, written from scratch. The model reads a short story and picks the most sensible of four endings.

Guessing gets 25%, so tiny models will score near that. It's useful for spotting a trend across sizes. A tiny sample of quiz questions for testing is in `tests/data/hellaswag_tiny.jsonl`.

Then speed. `gptlab/flops.py` counts the maths operations a model needs, and works out MFU: what share of the GPU's top speed we actually use. Its formula is checked against PyTorch's own counter.

`gptlab/bench.py` times training for many setups and model sizes, and finds the biggest batch that fits in memory. Its settings are in `configs/bench/smoke.yaml`.

Tests: `tests/test_evaluate.py`, `tests/test_flops.py` and `tests/test_bench.py`.

> **📁 Files we just created**
> - `gptlab/evaluate.py`: test loss, perplexity, bits per byte and sample text.
> - `gptlab/hellaswag.py`: the HellaSwag quiz.
> - `tests/data/hellaswag_tiny.jsonl`: a tiny quiz sample for tests.
> - `gptlab/flops.py`: counts the maths and measures GPU use.
> - `gptlab/bench.py`: times training for many setups.
> - `configs/bench/smoke.yaml`: settings for the speed test.
> - `tests/test_evaluate.py`, `tests/test_flops.py`, `tests/test_bench.py`: the measuring tests.

Let's circle back. We have data, a model, a training loop that stops in time, safe saving and ways to measure. The machine is complete. Now we need to run it across many free sessions that keep switching off.

## Step nine: storage that survives sessions

Each Kaggle session starts empty. So where do the data and checkpoints live between sessions? On Hugging Face.

`gptlab/hub.py` uploads and downloads them. But there's a catch: every saved version counts towards storage. So it only keeps the `latest` and `final` checkpoints, and it "squashes" the history down to one snapshot.

The downside? Old checkpoints are gone for good.

## Step ten: the job system

Now the part that ties it all together.

`runs/queue.yaml` is the to-do list. Here's what it says right now: a session lasts 11 hours, and training stops 20 minutes early to save. There are two jobs: `smoke`, which checks the whole pipeline on GPU, and `data-fineweb-edu-16k`, which builds the real dataset on a free CPU session. Experiments get added after the smoke test passes.

`gptlab/runner/queue.py` reads that list and picks the next job that can run. It skips jobs that are done, waiting on another job, or need a different kind of computer. And it skips jobs another session is working on, unless that session has gone quiet for 30 minutes, which means it probably died.

`gptlab/runner/results.py` saves each job's status and logs to a separate git branch called `results`. If two sessions save at the same moment, one simply re-fetches and tries again.

`gptlab/runner/run.py` is the runner itself. It claims a job, downloads the data and latest checkpoint, trains with a background uploader, sends a "still alive" signal every 10 minutes, stops before the time limit, and pushes the logs. `gptlab/runner/__main__.py` lets you start it with `python -m gptlab.runner`.

And `gptlab/runner/ddp_check.py` is a quick check at the start of a GPU session that the two GPUs can talk to each other. If it fails, the runner tries a safer setting.

The smoke job has its own settings, `configs/smoke/smoke.yaml`. And here's a lovely detail: it deliberately stops the training after 100 steps, then resumes from the Hugging Face copy. So it tests the restart on real hardware.

Finally, `kaggle/runner.ipynb`, the notebook you paste into Kaggle once. It just starts the runner. All the logic lives in the project, so the notebook never needs to change.

Tests: `tests/test_queue.py`, `tests/test_results.py` and `tests/test_runner.py`. That last one is great: it pretends to be several Kaggle sessions in a row, using a local folder in place of Hugging Face and GitHub. Plus `tests/test_setup.py`, which checks the package imports, and `tests/conftest.py`, with shared test helpers. 193 tests pass on a normal computer.

> **📁 Files we just created**
> - `gptlab/hub.py`: stores data and checkpoints on Hugging Face, keeping only latest and final.
> - `runs/queue.yaml`: the to-do list of jobs.
> - `gptlab/runner/queue.py`: picks the next job that can run.
> - `gptlab/runner/results.py`: saves job status and logs to the `results` branch.
> - `gptlab/runner/run.py`: the runner that does each job.
> - `gptlab/runner/__main__.py`: lets you start the runner with one command.
> - `gptlab/runner/ddp_check.py`: checks the two GPUs can talk to each other.
> - `configs/smoke/smoke.yaml`: the smoke job's settings, including a deliberate stop and restart.
> - `kaggle/runner.ipynb`: the notebook pasted into Kaggle once.
> - `tests/test_queue.py`, `tests/test_results.py`, `tests/test_runner.py`: job system tests.
> - `tests/test_setup.py` and `tests/conftest.py`: package check and shared test helpers.

## So, how's it doing?

The first GPU test run, the smoke job, has now happened. The first attempt froze in the data step until Kaggle stopped it 12 hours later, using about 12 of the 30 weekly GPU hours. The fix: start helper processes fresh, fail loudly if one dies, and stop any job that goes silent for 30 minutes.

The second attempt, `smoke-2` on 7 October, passed every step in 12.5 minutes. It measured real speeds, from about 625,000 tokens a second for the smallest model to about 37,000 for the 97.5-million-parameter one. It showed restarting on a GPU isn't bit-for-bit identical, but the difference is no bigger than normal noise. And it found two bugs, both fixed: the quiz file had moved, and the data summary miscounted characters.

Those speeds went into a new file, `EXPERIMENTS.md`: the plan for five experiments, needing about 46 GPU hours. A new script, `scripts/make_configs.py`, writes all the experiment settings in `configs/exp/` from one table. The real experiments, and their loss and quiz scores, haven't run yet.

What *is* proven: 193 tests pass on a normal computer, including the exact-restart test and the batch-adding test.

## What's still missing?

- **No experiment has run yet**, so no loss or quiz results. Only the smoke test has run on a GPU.
- **Restarts on a GPU aren't bit-for-bit exact**, though the difference is within normal noise.
- **Only exact copies of pages are removed**, not near-copies.
- **Deleted checkpoints can't be recovered.**
- **`REPORT.md` doesn't exist yet.** It fills in as experiments finish.

## Let's put it all together

So let's look at it in one breath.

We set up **exact settings** that are saved in full with every run. We read **web text** in a shuffled order, **cleaned** it, and set aside **test text by fingerprint**, so it can't leak. We built a **tokenizer** and **compact shards** with a checked header. We wrote a **loader** where one step number marks the exact position.

Then the **model**, with swappable parts, and a **training loop** using both GPUs, with a **deadline stop**. **Safe checkpoints** save everything, and a test proves restarts are exact. We added ways to **measure** quality and speed.

Finally, **Hugging Face storage**, a **to-do list of jobs**, a **results branch**, and a **runner** that turns many short free sessions into one long job.

Notice how it links. The single step number from the loader is what makes the checkpoint restart exact. The deadline in the training loop comes from the session settings in the queue.

The smoke job tests the restart on real hardware. Every piece supports the main challenge: long training on computers that keep switching off.

That's the project. Everything's built and tested, the GPU test run passed, and the real experiments are next.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
