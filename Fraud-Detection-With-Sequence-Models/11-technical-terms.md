# Card Fraud with Sequence Models: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow the work: the fraud itself, the data, features, the models, training, measuring, deciding, and serving.

---

## 1. The Fraud

### Card testing
**What it means:** criminals make lots of tiny payments to check whether stolen card details work.

**Why it's needed here:** it's one of the three simulated attacks. It's all about the pattern, so it's where reading payments in order helps most. The GRU caught 92% of these, against 84% for LightGBM.

### Account takeover
**What it means:** a criminal gains control of a real customer's account and spends from it.

**Why it's needed here:** it's the second simulated attack. It shows up as a sudden break from the customer's normal behaviour.

### Merchant compromise
**What it means:** a shop's systems are hacked, and card details of its customers are stolen.

**Why it's needed here:** it's the third simulated attack. It links many customers through one shop, which a single customer's history can't fully see.

### Confounder
**What it means:** innocent behaviour that looks like fraud, like travelling abroad or a sudden large purchase.

**Why it's needed here:** without look-alikes, the task would be unrealistically easy. The simulation adds many, including 4% of fraud that nobody ever reports.

### Adversarial drift
**What it means:** fraudsters deliberately changing tactics to avoid being caught.

**Why it's needed here:** it's built into the last 15% of the data. It tests whether models keep working after criminals adapt.

---

## 2. The Data

### Synthetic data (simulation)
**What it means:** realistic, made-up data where the true answers are known.

**Why it's needed here:** public datasets don't say which attack caused each fraud. With a simulation, you can ask "which attack is this model blind to?"

### Class imbalance
**What it means:** when one outcome is far rarer than the other.

**Why it's needed here:** only about 0.65% of payments are fraud. Models can get lazy and just say "not fraud" every time, so special training methods are tested.

### Temporal split and embargo
**What it means:** a temporal split trains on earlier data and tests on later data. An embargo leaves a gap between them.

**Why it's needed here:** real models predict the future from the past. The one-day gap stops information leaking across the boundary.

### Data leakage
**What it means:** when a model accidentally learns from information it wouldn't have in real life, usually from the future.

**Why it's needed here:** it's the most dangerous silent bug in fraud work. The project's tests scramble or delete future data and check the scores don't change. A leak was found and fixed on the first run.

---

## 3. Features

### Feature
**What it means:** one fact about a payment, turned into a number.

**Why it's needed here:** LightGBM learns only from features. The sequence models get features too, alongside the raw order of payments.

### Causal feature
**What it means:** a feature built only from the past, never the future.

**Why it's needed here:** in real life, you only know what's already happened when a payment arrives.

### Velocity
**What it means:** how many payments, or how much money, in a recent time window, like "5 payments in the last 10 minutes".

**Why it's needed here:** a sudden burst of payments is a classic fraud sign.

### Z-score
**What it means:** how unusual a number is for this customer, measured in "typical spreads" from their average.

**Why it's needed here:** £500 is normal for some customers and very unusual for others. The z-score captures that.

### Target encoding (out-of-time)
**What it means:** replacing a category, like a shop, with its past fraud rate. "Out-of-time" means only using rates from earlier periods.

**Why it's needed here:** a shop's fraud history is useful. Using future rates would leak information, so only past ones are used.

### Online/offline parity
**What it means:** the live system ("online") builds features exactly the same way as training ("offline").

**Why it's needed here:** if they differ, the model sees different facts live than it learned from. Here, one piece of code is used for both, and a test checks they match.

### Bounded history
**What it means:** only looking back a fixed number of payments.

**Why it's needed here:** a live request can't carry a customer's whole history. Features use at most the previous 31 payments. The downside is that longer-term patterns are lost.

### Padding and masking
**What it means:** padding fills short lists with blank spaces so all lists are the same length. A mask tells the model which spaces are blank.

**Why it's needed here:** customers have different numbers of past payments. Every window is padded to 32 so the models can process them together.

---

## 4. The Models

### Baseline (LightGBM)
**What it means:** LightGBM builds many small yes/no flowcharts, each fixing the last one's mistakes. As a baseline, it's the standard to beat.

**Why it's needed here:** it's the usual choice for fraud. The project made it as strong as possible, so any win by a sequence model is meaningful.

### Rule floor
**What it means:** a few simple, hand-written rules, like "flag more than X payments in Y minutes".

**Why it's needed here:** it's the lowest bar. Any model worth using should beat it.

### Sequence model
**What it means:** a model that reads a list of events in order.

**Why it's needed here:** fraud patterns unfold over several payments. The project tests three types.

### GRU (gated recurrent unit)
**What it means:** a sequence model that reads one item at a time and keeps a running memory, deciding what to remember and forget.

**Why it's needed here:** it was the winner. It's also smaller and faster to train than the Transformer.

### TCN (temporal convolutional network)
**What it means:** a sequence model that slides small pattern-detectors along the sequence, looking further back at each layer.

**Why it's needed here:** it's a second way of reading sequences, which makes the comparison fair.

### Transformer
**What it means:** the design behind ChatGPT. It lets each item look at every earlier item at once, using "attention".

**Why it's needed here:** it's the third type tested. With only about 1,100 fraud examples to learn from, it didn't have enough data to shine.

### Embedding
**What it means:** a short list of numbers learned for each category, like each shop, capturing how they behave.

**Why it's needed here:** it lets the models use categories like shops. Sizes are capped at 16 numbers, and rare shops share one entry, so the model doesn't memorise noise.

### Parameters
**What it means:** the adjustable numbers inside a model that training tunes.

**Why it's needed here:** they measure model size. The GRU has about 78,000 and the Transformer about 106,000.

---

## 5. Training

### Loss function (BCE, focal)
**What it means:** the score training tries to reduce. BCE is the standard one for yes/no problems. Focal loss puts extra weight on hard examples.

**Why it's needed here:** with rare fraud, the choice of loss matters. Both are tested.

### Weighted sampling
**What it means:** showing rare examples more often during training.

**Why it's needed here:** it's the third way tested for handling rare fraud. It shows fraud about 40 times more often.

### Early stopping
**What it means:** stopping training when results on held-out data stop improving.

**Why it's needed here:** it prevents memorising the training data. Here, it watches PR-AUC, the measure that matters for fraud.

### Seeds
**What it means:** starting numbers for randomness. Different seeds give slightly different results.

**Why it's needed here:** every model is trained 3 times with different seeds, and results are shown as averages with a spread. That shows whether a win is real or luck.

---

## 6. Measuring

### PR-AUC
**What it means:** a score for how well a model finds rare cases without too many false alarms. From 0 to 1, higher is better.

**Why it's needed here:** with rare fraud, it's far more useful than accuracy. The GRU scored 0.87 and LightGBM 0.83.

### Precision and recall at capacity
**What it means:** precision is how many alerts were real fraud. Recall is how much fraud was caught. "At capacity" means when you can only review a fixed share of payments, like 0.5%.

**Why it's needed here:** fraud teams can only check so many alerts. This measures performance at realistic workloads.

### Per-mechanism recall
**What it means:** how much of each attack type was caught.

**Why it's needed here:** a model might catch card testing but miss account takeovers. This shows exactly where each model is blind.

### Calibration
**What it means:** whether the model's chances match reality.

**Why it's needed here:** the cost rule needs honest chances. The project checks calibration especially in the "alert zone", where decisions happen.

### Temperature scaling and isotonic regression
**What it means:** two ways to adjust a model's chances to match reality. Temperature scaling softens or sharpens all scores. Isotonic uses a step shape.

**Why it's needed here:** focal loss made the models under-confident. Isotonic fixed calibration without hurting the ranking.

---

## 7. Deciding and Serving

### Cost policy
**What it means:** choosing a cut-off that gives the lowest expected cost.

**Why it's needed here:** a false alarm is assumed to cost £8, and a missed fraud £15 plus the amount. The cut-off is picked on validation data, then tested. The £8 is also varied from £2 to £30 to check the results hold.

### ONNX
**What it means:** a standard file format for AI models, which a fast tool (ONNX Runtime) can run.

**Why it's needed here:** it made the model's calculation about 2.8 times faster than running it in the training tool.

### Latency (p99)
**What it means:** latency is how long an answer takes. p99 means 99% of answers are this fast or faster.

**Why it's needed here:** card payments need quick answers. The budget is 50 ms, and the p99 is about 7 ms. Building features takes longer than running the model.

### Audit log
**What it means:** a permanent record of every decision.

**Why it's needed here:** each decision is written before replying, so it can always be traced.

### PSI (population stability index)
**What it means:** a common alarm that measures how much the spread of scores has shifted.

**Why it's needed here:** it never fired, even when fraudsters changed tactics. That's an important lesson about relying on it alone.

### Alert-rate monitoring
**What it means:** watching how often alerts fire at a fixed cut-off.

**Why it's needed here:** it caught the change in tactics in the first affected time window, when PSI didn't.
