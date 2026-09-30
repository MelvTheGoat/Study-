# Stock Question Agent: System Design for Beginners

This project plans an AI helper that answers factual questions about US and Nigerian stocks, and, just as importantly, a way to measure how often it's right. It's at an early stage: the "plumbing" for talking to the AI is built and tested, but the AI helper itself isn't built yet. That makes it a great example of building the measuring stick *before* the thing you measure.

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Model**: the AI program itself, which has learned patterns from lots of text.
- **Agent**: an LLM that can use tools, like a price lookup or a calculator, to finish a task.
- **Evaluation (eval)**: a test of the AI using questions whose correct answers you already know. Like an exam with an answer sheet.
- **Cache**: a saved copy of an answer, so you don't have to ask again.
- **Retry**: trying a request again after it fails, because some failures are temporary.
- **Config**: a settings file that describes one full test run.

Throughout, **Built** means it exists in the code today, and **Planned** means it's designed but not written yet.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The goal is to measure honestly, not just to build a clever bot. Models know a lot about Apple and very little about Nigerian Breweries. So testing both shows how much of the model's skill is memory, not reasoning.

**Step 2: Figure out the data.** We need stock prices and company filings with known correct answers. The official Nigerian exchange website blocks automated access, so finding allowed data is the real challenge.

**Step 3: Sketch the main parts.** A config describes the run, an agent asks the model questions, and graders mark the answers. Every call to the model goes through a stack of helpers.

**Step 4: Walk through one question.** Follow one question from the config to the model and back, checking where it's saved and logged.

**Step 5: Decide how to know it works.** Count how often answers are right, and record how long and how much each call took.

**Step 6: Plan for problems.** The free computer time is limited, calls can fail, and runs can be killed halfway. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Answer factual questions about **US** and **Nigerian (NGX)** stocks. *(Planned)*
- Run the model on a free **Kaggle** GPU (a special computer chip that runs AI fast), with about **30 hours** a week. *(Planned)*
- Never pay twice for the same question: save every answer. *(Built)*
- Retry temporary failures up to **4** times. *(Built)*

How long can retries take at most? Step by step:

1. The wait starts at 1 second and doubles each time: 1, 2, 4, 8 seconds.
2. 1 + 2 + 4 + 8 = **15 seconds** at most.
3. Each wait is also cut by a random amount (up to half), so it's often less.

### The Big Picture (Step 3)

```
   [Run Config]
        |
        v
   [Agent: asks questions]   (Planned)
        |
        v
   [Logger] --> [Cache] --> [Retry] --> [HTTP Client]   (Built)
                                            |
                                            v
                              [Model on free GPU]   (Planned)
        |
        v
   [Graders: mark answers]   (Planned)
```

Step by step:

1. A config file describes the run: which model, which questions, which kind of agent. *(Built)*
2. The agent turns each question into a request for the model. *(Planned)*
3. The Logger writes down the call: time taken, size, and whether it came from the cache. *(Built)*
4. The Cache checks if this exact request was asked before. If yes, it returns the saved answer straight away. *(Built)*
5. If not, Retry sends it on, and tries again if it fails in a temporary way. *(Built)*
6. The HTTP Client (the part that sends messages over the internet) talks to the model server. *(Built)*
7. Graders mark each answer as right or wrong. *(Planned)*

### The Main Parts (Step 3)

**Run Config.** One settings file describes a whole run, and it's checked strictly. A typo like "temparature" causes an error instead of being quietly ignored. It's like a recipe card that refuses to be used if an ingredient is misspelled. Each config also gets a short ID code, so you can tell whether two results came from the same setup.

**Logger.** It records every call: how long it took, how much text went in and out, and whether it was cached. It's like a till receipt for every question. Cost and speed are results too, not just accuracy.

**Cache.** Each request gets a unique fingerprint made from *all* its settings. If the same request comes again, the saved answer is returned. It's like keeping your marked homework, so re-checking it doesn't mean doing it again. Files are written safely, so a run killed halfway can't leave a broken half-file.

**Retry.** Some failures are temporary (the server is busy), and some are permanent (the request is wrong). It only retries the temporary kind. It's like redialling when the line is busy, but not when you've dialled a wrong number.

**Fake Clients for Tests.** Tests use a pretend model that gives scripted replies. One special fake fails the test if it's called at all. That proves some paths never reach the real model.

**Graders and Agent (Planned).** Graders will check numbers within a small tolerance, exact answers, and whether the source is right. The plan has three kinds of agent: model only, model plus given notes, and model plus tools.

The planned model is squeezed to 4-bit to fit on the free GPU (advanced - skip for now).

### How We Know It's Working (Step 5)

- **Tests**: 64 automatic checks pass in about 1.5 seconds. They run a second time with the internet blocked, to prove no test secretly uses it.
- **Accuracy**: the share of questions answered right. **Not measured yet**, because the question set doesn't exist.
- **Cost and speed per call**: will come from the Logger. Cached answers are marked, so they don't make the speed look better than it is.

### What Can Go Wrong (Step 6)

- **The data is blocked.** The official Nigerian exchange site uses a bot filter. The project refuses to sneak around it, and records every source it checked in a notes file.
- **The GPU time runs out.** With about 30 hours a week, wasted calls hurt. The cache means re-grading costs nothing.
- **A run is killed halfway.** Kaggle can stop sessions. Safe file writing means the cache is never left broken.
- **Results that wobble.** If answers change randomly, you can't spot small improvements. So the settings default to the most predictable mode.

## Quick Recap

- Build the measuring stick (the eval) before the thing you measure (the agent).
- Every model call goes through log, cache, retry and send, in that order.
- Strict configs catch typos before they mislabel results.
- Only temporary failures are retried, so no GPU time is wasted.
- Today the plumbing is built and tested, while the agent, data and graders are planned.
