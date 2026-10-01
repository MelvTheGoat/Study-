# Card Fraud with Sequence Models: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: the results come from a realistic **made-up** world of card payments, and the numbers are taken from the project's own report. The saved result files aren't included in the project, so checking them means re-running the experiment.

## 1. The Problem

Card fraud often doesn't show in a single payment. It shows in the *pattern*. Nine tiny payments in four minutes. A sudden break from how a customer normally spends.

Most fraud teams use a standard model called LightGBM, which looks at a table of facts about each payment. It's very hard to beat.

This project asks: can models that read a customer's recent payments *in order* do better? And do they keep working after fraudsters change their tactics?

## 2. The Big Idea

The big idea is "a fair, honest contest". The standard model is made as strong as possible, so any win by the new models really means something.

Every model is trained 3 times, and results are shown as averages with their spread. The winner is judged on money saved, not just on scores. Then the winner is served fast enough for real card payments.

## 3. How It Works, Step by Step

**Step 1: Build a made-up world.** It creates about 300,000 card payments from 2,500 customers over 180 days. About 0.65% are fraud, which is about 1 in every 150 payments.

**Step 2: Add three kinds of attack.** Card testing is lots of tiny payments to check stolen cards work. Account takeover is a criminal taking over a real account. Merchant compromise is a shop getting hacked.

**Step 3: Add innocent look-alikes.** Real customers also travel, make sudden big purchases, and use several devices. These are added so the task isn't unrealistically easy.

**Step 4: Make fraudsters adapt.** In the last 15% of the data, fraudsters change tactics. This tests whether models keep working.

**Step 5: Turn payments into facts.** For each payment, it works out facts like "how many payments in the last hour" and "how unusual is this amount for this customer". It only looks back at the previous 31 payments, never at future ones.

**Step 6: Train the contestants.** LightGBM, a simple rule set, and three sequence models (a GRU, a TCN and a Transformer). The sequence models read the last 32 payments in order.

**Step 7: Make the chances honest.** Each model's scores are adjusted so they match real fraud rates. This is called calibration.

**Step 8: Pick a cut-off by cost.** Blocking a real customer is assumed to cost £8. Missing a fraud costs £15 plus the payment amount. The cut-off is chosen to make the total cost as low as possible.

**Step 9: Serve the winner fast.** The winning model is saved in ONNX, a format that runs faster than the training tool. A web service scores each payment, records the decision, and replies.

**Step 10: Watch for changes.** Monitoring watches how often alerts fire, and other signals, to spot when fraudsters change tactics.

## 4. The Clever Parts

**The same code, training and live.** The code that builds facts for training is exactly the same code used in the live service. So the model sees the same kind of facts in both. A test checks they match.

**Tests that hunt for cheating.** "Leakage" means a model accidentally learns from the future. These tests scramble or delete future payments and check that scores don't change. A real leak was found and fixed on the first run.

**Known attack types.** Public fraud datasets don't say which attack caused each fraud. In the made-up world, they're labelled. So you can ask "which attack is this model blind to?"

**A better drift alarm.** A common alarm, called PSI, never went off when fraudsters changed tactics. But the alert rate dropped sharply right away. So watching the alert rate caught the change.

**Fast enough for card payments.** The full answer takes about 7 milliseconds at the slow end. The budget is 50 milliseconds. Building the facts takes longer than running the model itself.

## 5. The Important Words

- **Transaction**: one card payment.
- **Fraud**: a payment made by a criminal, not the real cardholder.
- **Model**: a program that learns patterns from past examples.
- **Sequence model**: a model that reads events in order, like reading a sentence word by word.
- **GRU**: a sequence model that reads one payment at a time and keeps a running memory.
- **Baseline**: a strong, standard model used for comparison, here LightGBM.
- **Feature**: one fact about a payment, as a number.
- **Leakage**: accidentally learning from information from the future.
- **Drift**: when behaviour changes over time.
- **Latency**: how long it takes to give an answer.

## 6. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **NumPy, pandas and pyarrow**: make and handle the payment data.
- **PyTorch**: builds the three sequence models.
- **LightGBM**: the strong standard model.
- **scikit-learn**: calibration and many measures.
- **SHAP**: explains LightGBM's decisions.
- **ONNX Runtime**: runs the winning model fast in the live service.
- **FastAPI**: the scoring service.
- **Docker Compose**: runs the service in a box.
- **pytest**: runs 144 automatic checks.

## 7. How Good Is It?

These come from the project's report, averaged over 3 training runs.

- **Overall**: the GRU scored 0.87 on PR-AUC, and LightGBM 0.83. PR-AUC measures how well a model finds rare fraud without too many false alarms, from 0 to 1.
- **Before tactics changed**: the two were tied.
- **After tactics changed**: the GRU held up much better, 0.77 against 0.62.
- **Card testing**: the GRU caught 92%, and LightGBM 84%. Here, the order of payments *is* the evidence.
- **Money**: the GRU was slightly cheaper per payment, but the difference was too small to be sure it's real.
- **The Transformer**: did worse than the smaller GRU. There were too few fraud examples for it to learn well.

## 8. What's Weak or Missing

- It's one made-up world, designed by the same person who built the models.
- 3 training runs aren't enough to prove the money difference.
- Only about 1,100 frauds were available for training.
- Fraud catches are counted per payment, not per attack, which flatters the results.
- Only looking back 31 payments loses longer-term patterns.
- The costs (£8 and £15) are assumptions.
- The result files aren't saved in the project.

## 9. What This Project Shows You Can Do

- Run a fair, honest contest between models.
- Hunt for leakage with tests that really try to break things.
- Keep training and live code identical.
- Choose decisions by money, not just by scores.
- Serve a model fast enough for real payments, and monitor it well.

## 10. Ten Things to Remember

1. It tests whether reading payments in order catches more fraud.
2. It uses about 300,000 made-up payments, with three labelled attack types.
3. Fraudsters change tactics in the last 15% of the data.
4. LightGBM is made as strong as possible, to keep the contest fair.
5. Facts only use the previous 31 payments, never the future.
6. The same fact-building code runs in training and live.
7. The GRU won overall, and held up much better after tactics changed.
8. The money difference was too small to be sure.
9. Answers take about 7 milliseconds, well under the 50 ms budget.
10. Watching the alert rate caught the change in tactics when PSI didn't.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
