# Card Fraud with Sequence Models: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: the results come from a realistic made-up world of card payments, and the numbers come from the project's own report. The saved result files aren't included in the project.*

---

## Okay, so what are we building?

Alright. Card fraud often doesn't show in one payment. It shows in the *pattern*.

Nine tiny payments in four minutes. A sudden break from how someone normally spends.

Most fraud teams use a model called LightGBM, which looks at a table of facts about each payment. It's very hard to beat. So here's our question: can models that read a customer's recent payments *in order* do better? And do they keep working after fraudsters change tactics?

So our project is a fair, honest contest. We make the standard model as strong as possible, so any win really means something. We judge on money, not just scores. And then we serve the winner fast enough for real card payments.

## So what do we need?

1. **Settings**, with every number that changes a result in one place.
2. **A made-up world of payments** with labelled attack types.
3. **One shape for every payment**, so real data can be swapped in later.
4. **Facts about each payment** that never see the future.
5. **Payment sequences** for the sequence models.
6. **A split by time**, with a gap.
7. **A strong standard model**, and three sequence models.
8. **A fair training process**, repeated with different seeds.
9. **Fair measures** for rare fraud.
10. **Honest chances**, and a **cut-off based on cost**.
11. **Explanations and monitoring.**
12. **A fast scoring service**, with a permanent record.

## Step zero: set up the workshop

First, a `README.md`, the front page, and a `.gitignore`. `pyproject.toml` describes the project and its tool settings.

Two install lists. `requirements.txt` is everything for training and experiments, including PyTorch and LightGBM. `requirements-serve.txt` is just what the live scoring service needs, which keeps that box small.

`Makefile` has a short command for each job: simulate, baseline, train, experiment, report, bench, serve, test, lint and typecheck. Two empty folders are kept with `.gitkeep` files: `data/raw/` for optional real data, and `results/` for experiment output.

The code lives in `src/`, and its `__init__.py` says what it's about: sequence models against gradient boosting, for real-time fraud.

Then `src/config.py`. Every setting that changes a reported number lives here, so the write-up and the code can't disagree. That includes the costs:

```python
false_positive_cost: float = 8.0
false_negative_cost_fixed: float = 15.0
```

Blocking a real customer costs £8. Missing a fraud costs £15 plus the payment amount. Hold onto those, they decide everything later.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `pyproject.toml`: project details and tool settings.
> - `requirements.txt`: tools for training and experiments.
> - `requirements-serve.txt`: just what the live service needs.
> - `Makefile`: one short command per job.
> - `data/raw/.gitkeep` and `results/.gitkeep`: keep the empty data and results folders.
> - `src/__init__.py`: marks the code folder as a package.
> - `src/config.py`: every number that changes a result, including the costs.

## Step one: a made-up world

Why made-up data first? Because public fraud datasets don't say *which kind of attack* caused each fraud. And that's exactly what we want to measure: which attack is a model blind to?

So `src/simulate.py` creates about 300,000 payments from 2,500 customers over 180 days. About 0.65% are fraud. And there are three labelled attacks: card testing (lots of tiny payments), account takeover, and a hacked shop.

But here's the important bit: it also adds **innocent look-alikes**. Travel, sudden big purchases, several devices, and even attackers using the victim's own device. Without them, the task would be too easy to mean anything.

And in the last 15% of the data, the fraudsters change tactics. That tests whether models keep working.

`src/data/schema.py` defines one shape for every payment. Real datasets get converted *into* that shape, so they can be swapped in.

`src/data/loaders.py` loads the made-up world by default. Two real datasets can be loaded too, but only if you download them yourself and ask for them specifically. `src/data/__init__.py` marks the data folder as a package.

Tested in `tests/test_simulate.py`. Its rule: the world must be hard enough to be worth measuring.

> **📁 Files we just created**
> - `src/simulate.py`: the made-up world, with three attacks, look-alikes and changing tactics.
> - `src/data/__init__.py`: marks the data folder as a package.
> - `src/data/schema.py`: one shape for every payment.
> - `src/data/loaders.py`: loads the made-up world, or real data on request.
> - `tests/test_simulate.py`: checks the world is hard enough.

## Step two: facts that never see the future

Now the facts about each payment, in `src/data/features.py`. And this file has two hard rules. A fact for a payment at a given moment can only read *earlier* payments. And the exact same code runs in training and in the live service.

The facts are things like "how many payments in the last hour", "how unusual is this amount for this customer", "how long since the last payment", and "is this a new shop for them". And it only ever looks back at the last 31 payments:

```python
DEFAULT_LOOKBACK = 31
```

Why a limit? Because a live request can't carry someone's entire history. So training uses the same limit as the live service. That way, the model always sees the same kind of facts.

Then `src/data/sequences.py` builds the windows for the sequence models: the last 32 payments, in order, with the payment being scored at the end. Shorter histories get padded with blanks, and a mask tells the model which spots are blank.

`src/data/splits.py` splits the data by time and nothing else. As its notes say, a random split would let a model train on someone's Thursday and test on their Wednesday. There's also a one-day gap between training and testing, so nothing leaks across.

## The most important test file

And now the tests. `tests/test_leakage.py`, and its notes say it plainly: "This is the file that matters most."

Why? Because a fact that quietly reads the future gives you an amazing score in testing, and a worthless model in real life. So these tests try to break things on purpose. They scramble future payments, delete future payments, flip labels, and check that the facts don't change at all.

There's also `tests/test_features_and_sequences.py` for everything else about facts and windows.

Let's pause. We have a hard, realistic world.

We have facts and windows that can't see the future, using the same code for training and live. And we have a fair time split. Now the contestants.

> **📁 Files we just created**
> - `src/data/features.py`: facts from only the last 31 payments, same code for training and live.
> - `src/data/sequences.py`: windows of the last 32 payments, padded and masked.
> - `src/data/splits.py`: a time split with a one-day gap.
> - `tests/test_leakage.py`: the tests that try hardest to break the no-future rule.
> - `tests/test_features_and_sequences.py`: other fact and window tests.

## Step three: the contestants

First, the champion to beat. `src/baselines.py` builds LightGBM on hand-made facts: payment speed, unusual amounts, time since last payment, new-shop flags.

Its notes say it's "built to win". A weak baseline proves nothing. There's also a floor of 4 simple rules, the lowest bar.

Then the sequence models, in `src/models/`. They all share one front end, in `src/models/base.py`, which turns each payment into numbers. It also keeps shop codes small: rare shops share one slot, so the model doesn't memorise noise. All three take the same input and give the same kind of output, so training treats them identically.

- `src/models/rnn.py`: a GRU, which reads one payment at a time and keeps a running memory. Its notes call it "the one most likely to be enough".
- `src/models/tcn.py`: a TCN, which slides small pattern-detectors along the sequence. It can't see the future by design.
- `src/models/transformer.py`: a small Transformer, which lets each payment look at every earlier one.

`src/models/__init__.py` marks the folder as a package. Tested in `tests/test_models.py`: shapes, masking, no future-peeking, and that each one can actually learn.

> **📁 Files we just created**
> - `src/baselines.py`: the strong LightGBM baseline, plus a simple rule floor.
> - `src/models/__init__.py`: marks the models folder as a package.
> - `src/models/base.py`: the shared front end for all sequence models.
> - `src/models/rnn.py`: the GRU.
> - `src/models/tcn.py`: the TCN.
> - `src/models/transformer.py`: the Transformer.
> - `tests/test_models.py`: model tests.

## Step four: fair training

`src/train.py` trains them all. And it makes three important choices.

First, it stops training based on PR-AUC, not the usual training score. With fraud this rare, that score can keep improving while the model gets no better at catching fraud.

Second, it tests three ways of handling rare fraud: a standard loss, a "focal" loss that focuses on hard examples, and showing fraud examples more often.

Third, every model is trained 3 times with different random starts, and results are shown as averages with their spread. That shows whether a win is real or just luck.

Tested in `tests/test_train.py`.

> **📁 Files we just created**
> - `src/train.py`: fair training, with three ways of handling rare fraud and three runs each.
> - `tests/test_train.py`: training tests.

## Step five: measuring fairly, and deciding by cost

`src/evaluation.py` measures. Its notes start with a warning: accuracy is meaningless here. At 0.65% fraud, a model that never flags anything scores 99.35% "accuracy". So it uses PR-AUC, precision and recall when you can only review a small share of payments, and how much of each attack type was caught.

`src/calibration.py` checks whether scores are real chances, or just rankings. That matters because the cut-off compares money, which needs honest chances. It pays special attention to the "alert zone", where decisions actually happen.

`src/policy.py` makes the decision. As its notes say, PR-AUC ranks models, but it doesn't tell you whether to deploy one.

Remember the £8 and £15? The cut-off is chosen to give the lowest total cost. It's picked on the validation data, then tested, separately for each run. And the £8 is varied from £2 to £30 to check the answer holds.

`src/artifacts.py` saves the scores each model gave, so calibration, policy, monitoring and the write-up all use exactly the same numbers.

Tests: `tests/test_evaluation.py` and `tests/test_calibration_and_policy.py`.

> **📁 Files we just created**
> - `src/evaluation.py`: fair measures for rare fraud.
> - `src/calibration.py`: checks scores are honest chances, especially in the alert zone.
> - `src/policy.py`: picks the cut-off by lowest cost.
> - `src/artifacts.py`: saves the scores everything else uses.
> - `tests/test_evaluation.py` and `tests/test_calibration_and_policy.py`: measuring and policy tests.

## Step six: explaining and watching

`src/explain.py` helps people understand the alerts. For the sequence models, it shows which earlier payments mattered most. For LightGBM, it uses SHAP, a method that shows how much each fact pushed the score. An analyst needs to know where to look.

`src/monitoring.py` watches for changes. And its notes make a sharp point: fraud drift isn't ordinary drift. Ordinary changes are accidental. Fraud changes are deliberate, because fraudsters are actively trying to dodge you.

And here's the proof. A common drift alarm, PSI, never fired when the fraudsters changed tactics. But the alert rate at a fixed cut-off dropped sharply, right away. `tests/test_monitoring.py` checks this end to end: train, change tactics, and check the alarm fires.

> **📁 Files we just created**
> - `src/explain.py`: shows which payments or facts drove an alert.
> - `src/monitoring.py`: watches for changes, especially deliberate ones.
> - `tests/test_monitoring.py`: proves the alarm fires when tactics change.

Let's circle back. We have a fair contest, fair measures, honest chances, a cost-based cut-off, explanations and an alarm. The science is done. Now let's run it all and serve the winner.

## Step seven: run the whole experiment

`src/experiment.py` runs everything in one go: train, evaluate, calibrate, decide costs, explain, monitor and prepare for serving. Its notes say running it produces every number in the README.

`src/report.py` turns those results into the README's tables, straight from the saved files, so the two can never disagree.

And `tests/test_metric_regression.py` is a safety gate. On a small made-up dataset, it fails the build if quality drops below a floor. The floor is deliberately loose, about half the real numbers, so it catches real breakage without failing on small wobbles.

> **📁 Files we just created**
> - `src/experiment.py`: runs the whole experiment.
> - `src/report.py`: builds the README's tables from saved results.
> - `tests/test_metric_regression.py`: fails the build if quality drops too far.

## Step eight: the fast scoring service

Now we serve the winner. A card payment needs an answer within a few hundred milliseconds. Our budget is 50.

`src/api/service.py` exports the model to ONNX, a format that runs faster than the training tool. It also runs the live fact-building, using the *same* code as training. `src/api/app.py` is the web service, built with FastAPI. It has addresses for scoring a payment, a health check, model details and live measurements.

A scoring request carries the payment plus 62 earlier payments. Why 62, when we only look back 31? So the oldest payment in the 32-slot window still has its own full 31-payment history, exactly as in training.

`src/api/schemas.py` defines what goes in and comes out. And `src/api/audit.py` records every decision before replying. As its notes say, a fraud system declines people's payments. When someone complains, you need the record.

`src/api/benchmark.py` measures speed three separate ways: the model alone, ONNX against PyTorch, and the full request. The full answer takes about 7 milliseconds at the slow end, and building the facts takes longer than running the model. `src/api/__init__.py` marks the folder as a package.

## The parity test

Tested in `tests/test_serving.py`. And its most important test is the parity test.

The classic way fraud systems fail in production is when the live facts differ from the training facts. So this test checks live and training facts are identical. It also checks the ONNX model gives the same answers as the original, and tests the API and the audit record.

Shared test setup is in `tests/conftest.py`, using only made-up data.

> **📁 Files we just created**
> - `src/api/__init__.py`: marks the service folder as a package.
> - `src/api/service.py`: ONNX export and the live fact-building.
> - `src/api/app.py`: the scoring web service.
> - `src/api/schemas.py`: what goes in and comes out.
> - `src/api/audit.py`: the permanent record of every decision.
> - `src/api/benchmark.py`: speed measurements, three ways.
> - `tests/test_serving.py`: the parity test, plus API and audit tests.
> - `tests/conftest.py`: shared test setup with made-up data.

## Step nine: packaging and checks

`Dockerfile` packs the scoring service into a box, called a container. `docker-compose.yml` runs it, with the trained model files attached read-only, so the service can't change them.

`.github/workflows/ci.yml` checks every change: lint, types and the tests, all on made-up data.

## Step ten: writing it all down

- `DECISIONS.md`: every big choice and why, like "make the baseline as strong as possible".
- `MODEL_CARD.md`: a standard summary of the model, its use and its limits.
- `CODE_WALKTHROUGH.md`: the project's own build-order walkthrough.
- `CODE_WALKTHROUGH.pdf`: the same walkthrough as a PDF.

> **📁 Files we just created**
> - `Dockerfile`: packs the scoring service.
> - `docker-compose.yml`: runs it, with model files attached read-only.
> - `.github/workflows/ci.yml`: automatic checks on every change.
> - `DECISIONS.md`: every big choice, explained.
> - `MODEL_CARD.md`: the model's summary and limits.
> - `CODE_WALKTHROUGH.md` and `CODE_WALKTHROUGH.pdf`: the project's own walkthrough.

## So, how's it doing?

These come from the project's report, averaged over 3 runs:

- **Overall**: the GRU scored 0.87 on PR-AUC, and LightGBM 0.83.
- **Before tactics changed**: they were tied.
- **After**: the GRU held up much better, 0.77 against 0.62.
- **Card testing**: the GRU caught 92%, and LightGBM 84%.
- **Money**: the GRU was slightly cheaper, but not by enough to be sure.
- **Speed**: about 7 milliseconds, well inside the 50 ms budget.

## What's still missing?

- **One made-up world**, designed by the same person who built the models.
- **3 runs aren't enough** to prove the money difference.
- **Only about 1,100 frauds** were available to train on.
- **Fraud is counted per payment**, not per attack, which flatters the results.
- **31 payments** of history loses longer-term patterns.
- **The results files aren't saved** in the project.

## Let's put it all together

So let's look at it in one breath.

We put every result-changing number in **one settings file**, including the costs. We built a **made-up world** with labelled attacks, innocent look-alikes and changing tactics. We made **facts that never see the future**, with the same code for training and live, and **tests that try hard to break that rule**.

We built a **strong baseline** and **three sequence models**, **trained them fairly** three times each, **measured fairly**, made the **chances honest**, and picked a **cut-off by cost**. We added **explanations** and an **alarm** that actually fires.

Then we ran it all in **one experiment**, served the winner through a **fast ONNX service** with a **permanent record**, and proved live and training facts are **identical**.

Notice how it links. The "same code, training and live" rule from step two is exactly what the parity test checks in step eight. The 31-payment limit is why requests carry 62 payments. And the £8 and £15 from step zero are what pick the cut-off in step five.

That's the project. A fair contest, judged by money, and served fast.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
