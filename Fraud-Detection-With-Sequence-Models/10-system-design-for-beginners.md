# Card Fraud with Sequence Models: System Design for Beginners

This project spots card fraud by looking at a customer's *recent pattern* of payments, not just one payment on its own. It honestly tests whether models that read payments in order can beat a strong, standard fraud model, especially after fraudsters change tactics. Then it serves the best one fast enough for real card payments. The data is a realistic simulation, where the type of every attack is known.

## Key Terms

- **Transaction**: one card payment.
- **Fraud**: a payment made by a criminal, not the real cardholder.
- **Model**: a program that learns patterns from past examples to make a guess.
- **Sequence model**: a model that reads a list of events *in order*, like reading a sentence word by word.
- **Baseline**: a strong, standard model used for comparison. Here it's LightGBM.
- **Feature**: one fact about a payment, as a number, like "payments in the last hour".
- **Drift**: when behaviour changes over time, like fraudsters changing tactics.
- **Latency**: how long it takes to give an answer.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Fraud often shows in a pattern, like nine tiny payments in four minutes. The question is whether reading the pattern in order beats the usual approach, and whether that saves money.

**Step 2: Figure out the data.** Public fraud datasets don't say which type of attack caused each fraud. So the project simulates payments with three labelled attack types, plus tricky innocent behaviour like travel and big one-off buys.

**Step 3: Sketch the main parts.** Build features, train several models, pick a cut-off based on costs, and serve the winner through a fast API.

**Step 4: Walk through one payment.** Follow one card payment from "request arrives" to "allow or flag", in under 50 milliseconds.

**Step 5: Decide how to know it works.** Measure how many frauds are caught, how many alerts are false, and the total cost.

**Step 6: Plan for problems.** Future information can leak into training, fraudsters adapt, and alarms can stay silent. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Learn from about **300,000** payments by **2,500** customers over **180** days.
- Handle fraud that is rare: about **0.65%** of payments.
- Answer each payment within **50 milliseconds**.
- Work well *after* fraudsters change tactics (the last 15% of the data).

How rare is the fraud? Step by step:

1. 0.65% means 0.65 in every 100 payments.
2. 300,000 × 0.65 ÷ 100 = **1,950** fraudulent payments.
3. So about **1 in 150** payments is fraud, and the model has only around 2,000 examples of fraud in total.

### The Big Picture (Step 3)

```
   [Training: compare models on simulated data]
                     |
                     v
   [Card payment + recent history]
                     |
                     v
   [Feature Builder: same code for training and live]
                     |
                     v
   [Sequence Model (GRU)] --> [Calibration: honest chance]
                                        |
                                        v
                             [Cost Rule: allow or flag]
                                        |
                                        v
                             [Audit Log] + [Monitoring]
```

Step by step:

1. Ahead of time, the training step compares a strong LightGBM model with three sequence models, and picks the winner.
2. A card payment arrives, together with the customer's recent previous payments.
3. The Feature Builder works out facts like "how many payments in the last hour", using exactly the same code as in training.
4. The Sequence Model reads the last 32 payments in order and gives a fraud score.
5. Calibration turns the score into an honest chance of fraud.
6. The Cost Rule compares the chance with a cut-off, and allows or flags the payment.
7. The decision is written to the Audit Log before replying, and Monitoring watches for changes.

### The Main Parts (Step 3)

**Simulated World.** It creates three attack types: card testing (many tiny payments), account takeover (a criminal takes over an account), and a hacked shop. It also adds innocent look-alikes, like travel and sudden big purchases. It's like a flight simulator with realistic bad weather: you know exactly what happened, so you can judge the pilot fairly.

**Feature Builder.** It looks only at the customer's previous 31 payments, never future ones. The *same* code runs in training and in the live service, so the model sees the same kind of facts in both. It's like using the same recipe in the test kitchen and in the restaurant.

**The Models.** LightGBM (a tool that builds many small yes/no flowcharts) is the strong baseline, built to win. It competes against three sequence models: a GRU, a TCN and a Transformer. The GRU reads payments one at a time and keeps a running memory, like a detective reading a diary page by page.

**Cost Rule.** Blocking a real customer is assumed to cost £8, and missing a fraud £15 plus the payment amount. The cut-off is chosen to make the total cost as low as possible. It's like a shop deciding how strict its security guard should be.

**Fast Serving.** The winning model is saved in ONNX, a format that runs faster than the training tool. The full answer takes about 7 milliseconds at the slow end, well inside the 50 ms budget.

**Monitoring.** It watches how often alerts fire at a fixed cut-off. When fraudsters changed tactics, the alert rate dropped sharply, which gave the warning.

How the TCN and Transformer read sequences differently from a GRU is a bigger topic (advanced - skip for now).

### How We Know It's Working (Step 5)

These come from the project's report, averaged over 3 training runs.

- **PR-AUC**: how well the model finds rare fraud without too many false alarms, from 0 to 1. The GRU scored **0.87**, and LightGBM **0.83**.
- **After tactics changed**: the GRU held up much better, scoring 0.77 against LightGBM's 0.62. Before the change, they were tied.
- **Cost per payment**: the GRU was slightly cheaper, but the difference wasn't big enough to be sure it's real.
- **Speed**: about 7 milliseconds at the slow end.

### What Can Go Wrong (Step 6)

- **Future information leaks in.** Tests scramble or delete future payments, and check that scores don't change.
- **Fraudsters adapt.** A common drift alarm (PSI) never fired. Watching the alert rate caught the change in the first affected time window.
- **Too few examples.** With about 2,000 frauds, bigger models like the Transformer can't learn enough.
- **One made-up world.** The same person designed the simulation and the models, so real-world results may differ.

## Quick Recap

- Fraud often shows in a pattern, so reading payments in order can help.
- A strong baseline makes the comparison honest.
- The same feature code runs in training and live, and it never sees the future.
- The cut-off is chosen by cost, not by accuracy.
- The sequence model held up better when fraudsters changed tactics.
