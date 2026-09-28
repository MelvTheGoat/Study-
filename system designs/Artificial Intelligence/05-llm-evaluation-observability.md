# LLM Evaluation and Observability

This system answers two questions about an AI app: "Is it good enough to release?" and "Is it still working well now that real people use it?" Every company with a chatbot, like a bank's support bot, needs both answers. It matters because AI apps can get worse quietly, and without checks you only find out when customers complain.

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **Evaluation**: testing the AI's answers *before* release, like an exam.
- **Observability**: watching the AI *after* release, using records of what it did. Like a car dashboard.
- **Test set**: a fixed list of questions with good answers, used to check quality. Also called a "golden set".
- **LLM-as-judge**: using a second AI to grade answers, like a teaching assistant marking exams.
- **Log**: a saved record of one request, including the question, answer, time taken and cost.
- **Hallucination**: when the AI confidently makes something up.
- **Alert**: an automatic message to the team when a number goes outside a safe range.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Decide what "a good answer" means for your app: correct, polite, on topic, and fast. Write it down in simple words, because every check you build will test these points.

**Step 2: Figure out the data.** Before release, you need test questions with good answers. After release, you need logs of real requests and user feedback, like thumbs-up and thumbs-down.

**Step 3: Sketch the two loops.** There's a *before release* loop (test every change) and an *after release* loop (watch real use). Bad real-world examples should flow back into the test set.

**Step 4: Walk through one change.** Imagine someone edits the prompt (the instructions sent to the AI). Follow the change from testing, to release, to monitoring.

**Step 5: Decide how to know it works.** Pick a handful of numbers to watch, like pass rate, hallucination rate, speed and cost.

**Step 6: Plan to keep improving.** Tests go stale and graders make mistakes. Plan regular reviews by real people.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design quality checks for a bank's support chatbot.

- A test set of **200** questions.
- A rule: a change can only be released if it passes at least **90%** of tests, and doesn't do worse than the current version.
- In real use: **10,000** conversations per day.
- Humans review a **1%** sample: 10,000 × 0.01 = **100 conversations per day**.

### The Big Picture (Step 3)

```
   BEFORE RELEASE
   [Change to prompt or model]
            |
            v
   [Run Test Set: 200 questions] --> [Graders: rules + AI judge]
            |
            v
   [Pass 90%?] -- no --> fix and try again
            | yes
            v
   AFTER RELEASE
   [Live chatbot] --> [Logs] --> [Dashboards + Alerts]
            |
            v
   [Bad examples added to Test Set]
```

Step by step:

1. Someone changes the prompt or swaps in a new model (the AI program that writes the answers).
2. The new version answers all 200 test questions.
3. Graders score each answer, using simple rules plus an AI judge.
4. If it passes 90% and isn't worse than before, it's released. If not, it goes back for fixing.
5. Once live, every request is saved as a log, and dashboards show the key numbers.
6. Alerts warn the team if something drops. Bad real examples are added to the test set, so the same mistake is caught next time.

### The Main Parts (Step 3)

**Test Set.** A list of real-looking questions, each with a good answer or a list of facts the answer must include. It's like a driving test route that every new driver takes, so results are fair to compare.

Mix in easy questions, tricky ones, and questions the bot should politely refuse.

**Graders.** Answers are checked in three ways, from cheapest to most expensive:

- **Simple rules**: does the answer mention the required fact ("£5 fee")? Is it under 200 words?
- **LLM-as-judge**: another AI reads the question and answer and gives a score, like "correct / partly correct / wrong". It's fast, but it can be wrong too, so check its grades against human grades now and then.
- **Human review**: people grade a small sample. They are the final word, like a head teacher checking the marking.

**Logs and Tracing.** Every real request is saved: the question, the prompt, the answer, the time taken and the cost. For multi-step apps, we also record each step, which is called tracing. It's like a flight's black box: when something goes wrong, you can replay exactly what happened.

Langfuse (a tool that stores and displays LLM logs and traces) is one popular option.

**Dashboards and Alerts.** Dashboards show the key numbers over time. Alerts send a message when a number crosses a line, for example "thumbs-down rate over 10%". It's like a smoke alarm: you don't stare at the ceiling, it tells you when to look.

**Feedback Loop.** When a real conversation goes badly, it becomes a new test question. Over time, the test set grows to cover the mistakes that matter most.

Grading whole conversations over many turns, not just single answers, is harder (advanced - skip for now).

### How We Know It's Working (Step 5)

- **Pass rate**: on the test set, 184 of 200 passed = 184 ÷ 200 = **92%**. That's above our 90% bar.
- **Hallucination rate**: the share of answers with made-up facts, from human or AI review. If 3 of the 100 reviewed chats had one, that's **3%**.
- **Thumbs-down rate**: from real users. For example, 400 thumbs-down out of 10,000 chats = **4%**.
- **Speed and cost**: average response time and cost per conversation. A "better" version that's twice as slow may not be better overall.

### What Can Go Wrong (Step 6)

- **Stale test set.** Users start asking about a new product, and the tests don't cover it. Add fresh real questions every month.
- **The AI judge is biased.** It may prefer long answers, for example. Compare its grades with human grades regularly, and fix its instructions.
- **Private data in logs.** Logs may contain names or account numbers. Hide personal details before saving, and limit who can read logs.
- **Too many alerts.** If alarms go off constantly, people start ignoring them. Alert only on numbers that truly need action.

## Quick Recap

- Evaluation checks quality *before* release. Observability watches quality *after* release.
- A test set is your fixed exam. Re-run it on every change, with a clear pass bar.
- Grade with simple rules, an AI judge and a sample of human reviews.
- Log every request, show key numbers on dashboards, and alert on real problems.
- Turn real failures into new test questions, so the same mistake doesn't happen twice.
