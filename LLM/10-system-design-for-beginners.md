# Training a Small LLM from Scratch: System Design for Beginners

This project builds everything needed to train small ChatGPT-style models from scratch, on free computers. It covers preparing the text, building the model, training it, testing it, and a job system that survives the free computers switching off. The code is written and tested, but it hasn't been run on a real GPU yet, so there are no results so far.

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text by predicting the next piece of text, like ChatGPT.
- **Parameter**: one of the adjustable numbers inside a model. Training tunes them. These models have about 1 million to 100 million.
- **Token**: a small piece of text, like a word or part of a word. Models read and write tokens, not letters.
- **Tokenizer**: the tool that splits text into tokens and turns each one into a number.
- **Training**: showing the model lots of text and nudging its parameters so it predicts the next token better.
- **Loss**: a score for how wrong the model's guesses are. Training tries to push it down.
- **GPU**: a special computer chip that does lots of maths at once. Training needs one to be fast.
- **Checkpoint**: a saved snapshot of the model partway through training, like a save point in a video game.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The goal is to learn how LLMs are trained, and to run real experiments, like "how does a bigger model change the loss?"

**Step 2: Figure out the data.** We need lots of clean, good-quality text. This project uses FineWeb-Edu, a free collection of educational web pages.

**Step 3: Sketch the main parts.** Text is cleaned, split into tokens and stored. Then a training loop teaches the model and saves checkpoints, and tests measure how good it is.

**Step 4: Walk through one training session.** Follow one session on a free GPU, from "pick a job" to "save and stop before time runs out".

**Step 5: Decide how to know it works.** Measure the loss on text the model never trained on, and score it on a standard quiz.

**Step 6: Plan for problems.** Free sessions end after about 11 hours, computers crash, and storage is limited. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Train models with about **1 million to 100 million** parameters.
- Use free Kaggle computers with **2** GPUs, each with **16 GB** of memory.
- Handle sessions that end after about **11 hours**.
- Prepare about **2.6 billion** tokens of training text.

How much space does the text need? Step by step:

1. Each token is stored as a number that takes **2 bytes**.
2. 2.6 billion tokens × 2 bytes = **5.2 billion bytes**, which is about **5.2 GB**.
3. It's stored in files of 100 million tokens each: 2.6 billion ÷ 100 million = **26 files**.

### The Big Picture (Step 3)

```
   [Web text: FineWeb-Edu]
            |
            v
   [Clean + split off test text]
            |
            v
   [Tokenizer: text to numbers]
            |
            v
   [Token files, stored online]
            |
            v
   [Job Runner] --> [Training Loop on 2 GPUs] --> [Checkpoints]
                               |
                               v
                         [Evaluation]
```

Step by step:

1. Web pages are read in a shuffled order, cleaned, and exact copies are removed.
2. 0.5% of pages are set aside as test text that the model never trains on.
3. The tokenizer turns the text into numbers, which are saved in files and uploaded to Hugging Face (a free website for storing AI models and data).
4. On a free GPU session, the Job Runner picks the next experiment from a to-do list.
5. The Training Loop downloads the data (and the last checkpoint, if resuming) and starts training.
6. It saves checkpoints regularly, and stops 20 minutes before the session ends so it has time to save.
7. When a job finishes, Evaluation measures how good the model is.

### The Main Parts (Step 3)

**Cleaning and Splitting.** It removes broken, too-short and duplicate pages, and records why each one was dropped. Which pages go to the test set is decided by a fingerprint of the page's text. So a page always lands in the same group, and a copy can't sneak into both. It's like sorting exam questions so none of the practice questions appear on the real exam.

**Tokenizer.** It learns the 16,384 most useful pieces of text from the training pages. A bigger list (32,000) would use up too much of a tiny model's parameters. It's like choosing a small, sensible dictionary for a beginner, instead of the giant one.

**The Model.** A GPT-style model built in PyTorch (a popular Python tool for building AI models). Many of its design choices can be switched on or off, so experiments can compare them fairly. It's like a car where you can swap the tyres or engine and test-drive each version.

**Training Loop.** It feeds the model text, measures the loss, and nudges the parameters to do better. It uses both GPUs at once, and it knows exactly which piece of text comes next from a single step number. So a stopped job can restart exactly where it left off, like a bookmark in a book.

Training in a shortened number format needs a "loss scaler" so tiny numbers don't vanish (advanced - skip for now).

**Checkpoints.** Each one saves the model, the training state, and even the random-number settings. It's written to a temporary file first, then renamed in one step. So a crash while saving can never ruin the last good save.

**Job Runner.** A to-do list of experiments lives in a file. Each session claims the next job, sends regular "still alive" signals, and records its status in a separate git branch (a separate line of saved files). It's like a relay race: each runner picks up the baton exactly where the last one stopped.

### How We Know It's Working (Step 5)

None of these have been measured yet, because there's been no GPU run.

- **Test loss**: the loss on the 0.5% of text the model never saw. Lower is better.
- **Bits per byte**: the same idea, but fair to compare across different tokenizers.
- **HellaSwag score**: a multiple-choice quiz about what happens next in a story. With 4 choices, guessing gets **25%**, and small models will sit near that.
- **Tests**: 170 automatic checks pass on a normal computer in about a minute. One proves that stopping and restarting gives exactly the same model as training straight through.

### What Can Go Wrong (Step 6)

- **The session ends mid-training.** Training stops 20 minutes early, saves, and marks the job "paused" so the next session can continue.
- **A crash while saving.** Safe saving means the last good checkpoint always survives.
- **Free storage fills up.** Only the latest and final checkpoints are kept, and old history is deleted, which means old saves can't be recovered.
- **Test text leaks into training.** The split uses the page's text fingerprint, and the tokenizer only learns from training pages.

## Quick Recap

- Clean the text, set some aside for testing, and turn it into tokens.
- Train on free GPUs, saving checkpoints safely and often.
- A single step number lets a job restart exactly where it stopped.
- A job to-do list plus a results branch turns short sessions into long experiments.
- Everything is built and tested, but no GPU run has happened yet.
