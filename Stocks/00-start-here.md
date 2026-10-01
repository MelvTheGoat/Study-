# Stock Question Agent: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: this project is at an early stage. Some parts are built and tested. Most of the main parts are only planned. This file tells you clearly which is which.

## 1. The Problem

AI chatbots can sound very sure of themselves and still be wrong. Ask one for a company's share price and it might make up a number that was never real. This is called a hallucination.

This project wants to build an AI helper that answers factual questions about stocks. It covers US stocks and Nigerian stocks (on the NGX, the Nigerian Exchange). Just as important, it wants to *measure* how often the helper is right.

## 2. The Big Idea

The big idea is "build the measuring stick first". Before building the clever AI helper, build the test that will grade it. Then every later change can be measured fairly.

There's also a clever twist. AI models have read a lot about Apple, and very little about Nigerian Breweries. So comparing the two shows how much of a model's skill is memory, and how much is real reasoning.

## 3. What's Built and What's Planned

**Built today:**
- A strict settings file that describes one full test run.
- The "plumbing" for talking to an AI model: logging, saving answers, and retrying failures.
- Pretend AI models for testing.
- 64 automatic tests, and automatic checks on every code change.

**Planned, not built yet:**
- The AI helper (the agent) itself.
- The set of test questions with known answers.
- The graders that mark answers right or wrong.
- The data collection for stock prices.
- Running the real AI model on a free computer.

So right now there are no results at all. The project's README honestly says "not run yet".

## 4. How It Will Work, Step by Step

**Step 1: Pick a settings file.** One file describes the whole run: which model, which questions, and which kind of helper. It's checked strictly, so a typo causes an error instead of being ignored. *(Built)*

**Step 2: Load the test questions.** Each question has a known correct answer, fixed to a certain date, because share prices change every day. *(Planned)*

**Step 3: Ask the AI.** The helper turns each question into a request for the model. *(Planned)*

**Step 4: The request goes through the plumbing.** First it's logged. Then the saved answers are checked. If this exact question was asked before, the saved answer comes back straight away. If not, the request is sent, and retried if it fails in a temporary way. *(Built)*

**Step 5: The model answers.** It's planned to run on a free Kaggle computer with a GPU (a special chip that runs AI fast). *(Planned)*

**Step 6: Mark the answers.** Graders check numbers within a small margin, exact answers, and whether the right source was used. *(Planned)*

**Step 7: Save the results**, labelled with a short code for the settings used. *(Planned)*

## 5. The Clever Parts (Already Built)

**Strict settings.** If you misspell a setting, like "temparature", most programs just ignore it. Your results would then be quietly mislabelled. This project refuses to run instead. Each settings file also gets a short ID code, so you can tell if two results came from the same setup.

**Never pay twice.** Free GPU time is limited to about 30 hours a week. So every answer is saved, keyed by a fingerprint of the *whole* request. Asking the same thing again costs nothing, and an answer made with different settings can never be reused by mistake.

**Safe saving.** Free sessions can be stopped at any moment. Answers are saved to a temporary file first, then renamed in one step. So a half-written file can never appear.

**Smart retries.** Some failures are temporary, like "server busy". Others are permanent, like a badly formed request. Only temporary ones are retried, waiting 1, 2, 4 then 8 seconds, so no GPU time is wasted.

**Honest speed numbers.** Saved answers are marked as saved. So they don't make the model look faster than it really is.

**Tests that prove things.** One pretend model fails the test if it's ever called. That proves some code paths never reach the real model. The tests also run a second time with the internet blocked.

## 6. The Important Words

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Hallucination**: when an AI confidently makes something up.
- **Agent**: an AI that can use tools, like a price lookup or calculator.
- **Evaluation (eval)**: testing the AI with questions whose answers you already know.
- **Grader**: code that marks one answer right or wrong.
- **Cache**: saved answers, reused instead of asking again.
- **Retry with backoff**: trying again after a failure, waiting longer each time.
- **GPU**: a special computer chip that runs AI fast.
- **Config**: a settings file that describes one test run.
- **Baseline**: a simple version to compare against, here "the model answering from memory alone".

## 7. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **Pydantic**: checks the settings strictly.
- **PyYAML**: reads the settings files.
- **httpx**: sends questions to the AI model.
- **pytest and pytest-socket**: run the tests, including with the internet blocked.
- **ruff**: tidies the code and spots small mistakes.
- **GitHub Actions**: runs the checks on every change.
- **vLLM (planned)**: will run the AI model fast.
- **Qwen2.5-7B (planned)**: the open AI model to be tested.
- **Kaggle (planned)**: free GPU time.

## 8. How Good Is It?

There are no results yet. How often the AI is right, how fast it is, and what it costs have **not been measured**. The question set doesn't exist yet.

What *is* proven: the plumbing works. All 64 tests pass in about 1.5 seconds.

## 9. What's Weak or Missing

- Most of the system doesn't exist yet: the agent, the questions, the graders and the data.
- The data is the real blocker. The official Nigerian exchange website blocks automated visitors, and the project refuses to sneak around that.
- Another possible data source has no terms of use page, so it isn't used yet.
- US data sources haven't been investigated yet.
- An earlier version of the project, built with different tools, was deleted to start again in Python.

## 10. What This Project Shows You Can Do

- Think about measurement *before* building.
- Build careful, reliable plumbing for calling AI models.
- Save money and computer time with smart caching and retries.
- Write tests that prove what code does *not* do.
- Respect data rules, even when it slows you down.

## 11. Ten Things to Remember

1. The goal is to measure how often an AI gets stock facts right.
2. It compares US and Nigerian stocks, to separate memory from reasoning.
3. The plan is to build the test before the AI helper.
4. Today, only the plumbing and tests are built.
5. Every model call goes: log, then saved answers, then retry, then send.
6. Strict settings turn typos into errors.
7. Saved answers mean re-grading costs nothing.
8. Only temporary failures are retried.
9. There are no results yet.
10. Finding allowed Nigerian stock data is the biggest challenge.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
