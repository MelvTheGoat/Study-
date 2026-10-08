# Training a Small LLM from Scratch: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: all the code is written and tested. The first real GPU test run passed on 7 October 2026, and the speeds are now measured. But the real training experiments haven't run yet, so there are no experiment results so far.

## 1. The Problem

Lots of people use AI chatbots like ChatGPT. Very few can explain how one is actually trained.

This project builds everything needed to train small ChatGPT-style models from scratch. It uses free computers and covers preparing the text, building the model, training it, testing it, and running long jobs.

## 2. The Big Idea

There are two big ideas.

**First: understand every step.** The training code is written directly in PyTorch, a popular tool for building AI models. It doesn't use a ready-made training framework that hides the details.

**Second: make free computers work for long jobs.** Kaggle gives free computers with 2 GPUs (special chips that run AI fast). But each session switches off after about 11 hours. So the project is designed to stop safely, save its progress, and carry on in the next session, without losing any work.

## 3. What It Will Do

It's set up to train models with about 1 million to 100 million parameters. A parameter is one of the adjustable numbers inside a model that training tunes. For comparison, ChatGPT-sized models have billions.

It's also set up for real experiments. For example, "how much better does a bigger model get?" or "which design choices help most?"

## 4. How It Works, Step by Step

**Step 1: Collect the text.** It uses FineWeb-Edu, a free collection of educational web pages. It reads about 2.9 million pages in a shuffled order.

**Step 2: Clean the text.** It removes broken pages, pages that are too short or too long, and exact copies. It records why each page was dropped.

**Step 3: Set some text aside for testing.** 0.5% of pages are kept aside and never used for training. A fingerprint of each page's text decides which group it goes in, so a page and its copy always land in the same group.

**Step 4: Build a tokenizer.** A tokenizer splits text into small pieces called tokens, and turns each one into a number. This one learns the 16,384 most useful pieces from the training text.

**Step 5: Save the numbers.** The text becomes about 2.6 billion tokens. Each token takes 2 bytes, so that's about 5.2 GB. It's saved in files of 100 million tokens each and uploaded to Hugging Face, a free website for storing AI models and data.

**Step 6: A free session picks a job.** A Kaggle session runs a job runner. It reads a to-do list of experiments and claims the next one it can run.

**Step 7: Train.** The training loop feeds the model text, measures how wrong its guesses are (the "loss"), and nudges its parameters to do better. It uses both GPUs at once.

**Step 8: Save often, and stop in time.** It saves checkpoints (snapshots of progress) regularly. 20 minutes before the 11-hour limit, it stops, saves, and marks the job "paused". The next session picks it up.

**Step 9: Test the finished model.** When a job finishes, it's measured on the held-back text and on a standard multiple-choice quiz.

## 5. The Clever Parts

**Restarting exactly where it stopped.** The data is read in a fixed shuffled order, and the position is just the step number. So one number tells a restarted job exactly which text comes next.

Even the random-number settings are saved. A test proves that stopping and restarting gives exactly the same model as training straight through. This has been proven on a normal computer, not yet on a GPU.

**Safe saving.** Each checkpoint is written to a temporary file first, then renamed in one step. A crash while saving can never ruin the last good save.

**A sensible word list.** A bigger list of 32,000 tokens would use up too much of a tiny model's parameters. So 16,384 was chosen.

**Swappable design choices.** Many parts of the model can be switched, like how it tracks word positions or which "thinking" layer it uses. That makes fair experiments easy, like swapping one car part and test-driving again.

**Working with old GPUs.** The free T4 GPUs don't support the safer short number format. So the project uses an older format plus a "loss scaler" that stops tiny numbers turning into zero.

**A results notebook in git.** Each job's status and logs are saved to a separate git branch (a separate line of saved files). Each running job sends a "still alive" signal every 10 minutes. If a job goes silent for 30 minutes, it's freed for another session.

**Staying inside free storage.** Only the latest and final checkpoints are kept on Hugging Face, and old history is deleted. The downside is that old saves can't be recovered.

## 6. The Important Words

- **LLM (large language model)**: an AI that reads and writes text by predicting the next piece.
- **Parameter**: one adjustable number inside a model.
- **Token**: a small piece of text, like a word or part of a word.
- **Tokenizer**: the tool that splits text into tokens and numbers them.
- **Training**: showing the model lots of text and nudging its parameters to improve.
- **Loss**: a score for how wrong the model's guesses are. Lower is better.
- **GPU**: a special chip that does lots of maths at once.
- **Checkpoint**: a saved snapshot of training, like a save point in a video game.
- **Validation text**: text kept aside to test the model fairly.
- **Perplexity and bits per byte**: two other ways to show how good the model is at predicting text.

## 7. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **PyTorch**: builds and trains the model.
- **tokenizers**: builds the tokenizer very fast.
- **pyarrow**: reads the web text files.
- **NumPy**: reads the huge token files a bit at a time.
- **Hugging Face Hub**: free storage for data and checkpoints.
- **git results branch**: stores each job's status and logs.
- **Kaggle**: free GPU sessions to run the jobs.
- **pytest**: runs 193 automatic checks on a normal computer.

## 8. How Good Is It?

The first GPU test run (a "smoke test" on a tiny slice of data) passed every step in 12.5 minutes on 7 October. The first attempt didn't: it froze in the data step until Kaggle stopped it 12 hours later, using up about 12 of the 30 free weekly GPU hours. The fix was to start helper processes fresh, and to stop any job that goes silent for 30 minutes.

The smoke test also measured real speeds. The smallest model trains at about 625,000 tokens a second, and the 97.5-million-parameter model at about 37,000.

Using both GPUs is 1.74 times faster than one. The full experiment plan now needs about 46 GPU hours. But the real experiments, and their loss and quiz scores, are still waiting.

What *is* proven: 193 automatic tests pass on a normal computer. They include the exact-restart test, and a check that adding up small batches gives the same result as one big batch. On the real GPUs, restarting isn't bit-for-bit identical, because GPU maths adds numbers in a varying order. But the difference a restart makes is no bigger than the difference between two identical runs.

When it runs, expect the tiny models to score close to 25% on the quiz. That's the same as guessing, since there are 4 choices. The quiz is useful for spotting a trend across model sizes, not as a final score.

## 9. What's Weak or Missing

- No training experiment has run yet, so no experiment results. Only the smoke test has run on a GPU.
- Exact restarts are proven on a normal computer, but not yet on a GPU, where small differences can creep in.
- Only exact copies of pages are removed, not near-copies.
- Deleting old checkpoints saves space but means there's no going back.
- Two planned report documents are mentioned but don't exist yet.

## 10. What This Project Shows You Can Do

- Explain and build every step of training a language model.
- Prepare a large text dataset carefully, without leaks between training and testing.
- Design long jobs that survive computers switching off.
- Squeeze real work out of free hardware.
- Write tests that prove tricky things, like exact restarts.

## 11. Ten Things to Remember

1. It trains small ChatGPT-style models from scratch.
2. Models range from about 1 million to 100 million parameters.
3. The text comes from FineWeb-Edu, cleaned and with 0.5% set aside for testing.
4. The tokenizer knows 16,384 pieces of text.
5. There are about 2.6 billion tokens, taking about 5.2 GB.
6. Free Kaggle sessions end after 11 hours, so training stops 20 minutes early and saves.
7. One step number lets a job restart exactly where it stopped.
8. Checkpoints are saved safely, so a crash can't ruin them.
9. A to-do list of jobs plus a results branch connects many short sessions.
10. Everything is built and tested, the GPU smoke test passed, and the real experiments are next.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
