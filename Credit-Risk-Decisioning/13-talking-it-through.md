# Loan Approval Decisions: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: the results come from a realistic made-up loan book, not real customers. That's on purpose, and I'll explain why.*

---

## Okay, so what are we building?

Alright. Someone applies for a loan. The lender has to decide: approve or decline?

Most people think the job is "predict who won't pay back". But a real lender needs more than that.

It needs four things. An **honest chance** that this person won't pay back. A **decision based on money**: what a bad loan costs and what a good one earns.

A **clear reason** for every decline, because the law often requires it. And **proof the decisions are fair** to different groups.

So that's our project. And the headline result isn't an accuracy score. It's **money**.

## So what do we need?

1. **Settings**, with every business assumption in one place.
2. **A made-up loan book** where the truth is known for everyone.
3. **A real-data path**, for when real data is available.
4. **Models**: a classic scorecard, and two others to compare.
5. **Honest chances**, through calibration.
6. **Fair measuring**, for a problem where defaults are rare.
7. **A cost-based cut-off.**
8. **Studies**: guessing what declined people would have done, and fairness.
9. **Reasons** for every decline.
10. **Monitoring** for changes over time.
11. **A safe model package**, so the wrong parts can't be paired.
12. **One decision engine**, an API, batch scoring and a permanent record.

## Step zero: set up the workshop

First, a `README.md`, the front page, and a `.gitignore`. `pyproject.toml` describes the project, its tools and its test settings.

The code lives in `src/`. Its `__init__.py` sums it up: calibrated chances of default, turned into decisions under unequal costs, with every decision explained.

Then `src/config.py`, and it matters a lot here. Every number that's a business judgement lives in this one file, not buried in the code. Here are the two big ones:

```python
lgd: float = 0.75
margin_rate: float = 0.09
```

A bad loan loses 75% of the money. A good loan earns 9%. Keep those two numbers in mind, they drive the whole decision.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `pyproject.toml`: the project's details and tool settings.
> - `src/__init__.py`: marks the code folder as a package, with a summary.
> - `src/config.py`: every business assumption in one place.

## Step one: a loan book where we know the truth

Here's the problem with real loan data. A lender only learns whether *approved* people paid back. Declined people never got the loan, so you never find out what they would have done. That makes two important claims impossible to check: whether bias corrections work, and whether decisions are fair.

So we make our own loan book. `src/simulate.py` creates 30,000 realistic applications over 36 months, with hidden rules for who defaults. And we keep the truth for *everyone*, including people who were declined.

It also plants some realistic problems on purpose. From month 24, customers start behaving differently. One group's income is recorded 15% too low, to test the fairness audit. And there's a hidden loan-officer judgement that affects who got approved, which is what makes bias correction really hard.

It's like a flight simulator. You can test dangerous situations safely, because you know exactly what really happened.

Tested in `tests/test_simulate.py`. Its notes say the generator is "the measuring instrument for everything else". If the planted bias isn't really there, every later result is meaningless.

## The real-data path

There's also `src/ingest.py`, for a real public dataset: 30,000 credit card accounts from the UCI archive. It downloads it, checks a fingerprint, and checks the columns.

But the download was blocked when this was built, so it's never been run on the real file. `tests/test_ingest.py` tests it with no internet, using a small stand-in.

> **📁 Files we just created**
> - `src/simulate.py`: a made-up loan book where the truth is known for everyone.
> - `tests/test_simulate.py`: checks the planted problems are really there.
> - `src/ingest.py`: the real-data path, not yet run on the real file.
> - `tests/test_ingest.py`: tests the real-data path offline.

## Step two: the models

First, the classic. `src/scorecard.py` builds a points scorecard, which is what banks actually use.

It groups each answer, like income, into ranges, scores each range by how risky it is, and adds up points. You can check it in a spreadsheet. It's the benchmark every other model has to beat.

Then `src/models.py` adds two more: a simple logistic regression, and LightGBM, a tool that builds many small yes/no flowcharts. All three sit behind one common shape, so everything later treats them the same.

And we split the data by *time*, not randomly. Learn from months 0 to 19, fine-tune on months 20 to 23, and test on months 24 to 35, where customers change. That's how a real lender works: build on the past, use on the future.

Tested in `tests/test_scorecard.py`.

> **📁 Files we just created**
> - `src/scorecard.py`: the classic points scorecard.
> - `src/models.py`: logistic regression and LightGBM, behind one shape.
> - `tests/test_scorecard.py`: scorecard tests.

## Step three: honest chances

Now, here's something people often skip. A model can rank people perfectly and still give wrong chances. And for lending, the chance itself matters, because expected loss is chance times loss times the amount.

So `src/calibration.py` checks the chances against reality and adjusts them if needed. It tries three options: no change, Platt (a smooth curve) and isotonic (a step shape). It picks the one whose chances line up best on later data. On this loan book, the raw LightGBM needed no change.

And there's a safety net: `tests/test_calibration_gate.py`. It compares calibration with a saved baseline in `tests/calibration_baseline.json`. If a code change quietly makes the chances worse, the build fails. Its notes call it a "specific and quiet" failure, the kind you'd never spot from accuracy alone.

Then `src/evaluation.py` measures fairly. Its notes start bluntly: accuracy is meaningless here. Only about 17% of borrowers default, so always saying "won't default" is right 83% of the time. So it uses better measures, like AUC, and precision when you can only review a set share of applications.

Tests: `tests/test_calibration.py` and `tests/test_evaluation.py`.

> **📁 Files we just created**
> - `src/calibration.py`: makes chances honest, picking the best method.
> - `tests/calibration_baseline.json`: the saved calibration baseline.
> - `tests/test_calibration_gate.py`: fails the build if calibration gets worse.
> - `src/evaluation.py`: fair measures for rare defaults.
> - `tests/test_calibration.py` and `tests/test_evaluation.py`: calibration and evaluation tests.

## Step four: the cut-off, from money

Now we turn a chance into a decision. `src/policy.py`, and its notes say it straight: the headline result is a cost, not an AUC.

Remember those two numbers? A good loan earns 9%, a bad one loses 75%. Here's the real formula, written in a comment in the code:

```python
p*  =  margin / (margin + LGD)  =  0.09 / 0.84  =  0.1071
```

So approve if the chance of default is below about 11%, and decline if it's above. It also checks the formula against actually trying every cut-off, and shows how the cut-off moves if the assumptions change.

Tested in `tests/test_policy.py`. Its central check is that the formula and the try-every-cut-off method agree.

Let's pause. We have a loan book with known truth, three models, honest chances, fair measures and a cut-off based on money. That's the core decision, done. Now the harder questions.

> **📁 Files we just created**
> - `src/policy.py`: turns a chance into approve or decline, using costs.
> - `tests/test_policy.py`: checks the formula and the sweep agree.

## Step five: the hard studies

**Guessing what declined people would have done.** `src/reject_inference.py` tries four methods to correct for only seeing approved people. And here's the honest finding: they made things worse in one setup and better in another. And a real lender can't tell which setup they're in.

Tested in `tests/test_reject_inference.py`, which checks the *findings* themselves. So if a change quietly reverses a conclusion, the build fails.

**Fairness.** `src/fairness.py` measures how different groups are treated. And its notes are clear: it doesn't quietly "fix" anything. It measures, and it shows the choice.

It found one group approved at 0.755 times the rate of another, below the common 0.8 warning line. It also showed the planted income error pushed that group's predicted risk up by about 2 points.

Tested in `tests/test_fairness.py`. Its rule: the audit must actually find the bias that was planted. An audit that says "no problems" on a book known to have one is broken.

> **📁 Files we just created**
> - `src/reject_inference.py`: four ways to guess what declined people would have done.
> - `tests/test_reject_inference.py`: checks the findings, not just the code.
> - `src/fairness.py`: measures fairness and shows the trade-offs.
> - `tests/test_fairness.py`: checks the audit finds the planted bias.

## Step six: reasons and monitoring

**Reasons.** `src/explain.py` does two different jobs. One explains the model overall, using SHAP, a method that shows how much each fact pushed a prediction. The other gives each declined applicant their reasons, based on where they lost the most scorecard points. That's simple and gives the same answer every time, which matters for a legal notice.

And reasons based on a protected trait, like age, are flagged for legal review instead of just being printed. Tested in `tests/test_explain.py`.

**Monitoring.** `src/monitoring.py` watches for changes over time. And its notes say something important: a credit model doesn't fail loudly. It fails by being quietly wrong.

Here's the proof: a common drift alarm stayed quiet while defaults rose 75%. Only a combined check, which also watches real default rates by month, caught it.

Tested in `tests/test_monitoring.py`.

> **📁 Files we just created**
> - `src/explain.py`: overall explanations, and reasons for each decline.
> - `tests/test_explain.py`: checks reasons are right and legally safe.
> - `src/monitoring.py`: drift and default-rate watching, with review triggers.
> - `tests/test_monitoring.py`: monitoring tests.

## Step seven: running all the experiments

`scripts/run_experiments.py` runs the whole programme. It creates the loan book, trains the models, runs every study, writes every table and chart, and builds the final model package. `scripts/__init__.py` marks the scripts folder as a package.

Everything it produces lands in `reports/`. The charts are in `reports/figures/`:

- `calibration_reliability.png`: predicted chances against real rates.
- `cost_frontier.png`: profit against approval rate.
- `fairness_tradeoff.png`: fairness against profit.
- `vintage_curves.png`: default rates by month of lending.

And the tables are in `reports/tables/`, numbered by study:

- **00**, the loan book: `00_simulation_summary.csv`.
- **01**, the scorecard: `01_information_values.csv`, `01_scorecard_coefficients.csv`, `01_scorecard_points.csv`.
- **02**, ranking before calibration: `02_discrimination_uncalibrated.csv`, `02_scorecard_gap_decomposition.csv`.
- **03**, calibration: `03_calibration_comparison.csv`.
- **04**, ranking after calibration: `04_discrimination_calibrated.csv`, `04_gains_table.csv`, `04_precision_at_capacity.csv`, `04_resampling.csv`.
- **05**, the cut-off: `05_approval_frontier.csv`, `05_cost_sensitivity.csv`, `05_policy_cost_comparison.csv`, `05_threshold_rules.csv`.
- **06**, guessing declined outcomes: `06_reject_inference.csv`, `06_uplift_sensitivity.csv`.
- **07**, fairness: `07_disparity_decomposition.csv`, `07_fairness_by_group.csv`, `07_fairness_summary.csv`, `07_fairness_tradeoff.csv`.
- **08**, explanations: `08_adverse_action_notice.txt` (an example decline letter), `08_shap_global_importance.csv`, `08_worked_example.json`.
- **09**, monitoring: `09_feature_drift.csv`, `09_score_distribution.csv`, `09_triggers.csv`, `09_vintage_curves.csv`, `09_vintage_summary.csv`.
- And `run_summary.json`, a summary of the whole run.

> **📁 Files we just created**
> - `scripts/run_experiments.py`: runs every study and builds the model package.
> - `scripts/__init__.py`: marks the scripts folder as a package.
> - `reports/figures/`: the 4 charts.
> - `reports/tables/`: the 33 result tables, numbered by study, plus `run_summary.json`.

Let's circle back. We've built the brain, run every study, and saved the results. Now we need to serve real decisions, safely.

## Step eight: a model package you can't mix up

Here's a quiet danger. A model and its calibration are two separate files. If you pair a model with the wrong calibration, it still ranks people well, so nothing looks wrong. But its chances are wrong, and so is every decision.

So `src/artifacts.py` packages them together. Every file from one training run carries the same ID and fingerprints. And when the service starts, if they don't match, it refuses to start, with an error built just for this.

Tested in `tests/test_artifacts.py`. Its notes say it checks "a safety property, not a feature".

> **📁 Files we just created**
> - `src/artifacts.py`: packages the model and calibration, and refuses mismatches.
> - `tests/test_artifacts.py`: tests the mismatch guard.

## Step nine: one decision engine

`src/engine.py` is the decision engine. It's the *only* code that turns an application into a decision. It loads the package, predicts the chance, works out the expected loss, applies the cut-off, and gives the reasons.

Why only one? Because both the live service and overnight batch scoring use it. If they had separate code, they could drift apart and give the same person two different answers. The most important test in `tests/test_engine.py` checks the two give *identical* results.

`scripts/score_batch.py` scores a whole file of applications through that same engine. Real credit teams do both: one at a time during the day, and the whole book overnight.

> **📁 Files we just created**
> - `src/engine.py`: the one decision engine.
> - `tests/test_engine.py`: checks live and batch decisions are identical.
> - `scripts/score_batch.py`: scores a whole file through the same engine.

## Step ten: the decision service

Now the service. `src/api/main.py` is built with FastAPI, a tool for building APIs. It has three addresses: one for decisions, one for a health check, and one describing the model.

`src/api/schemas.py` checks every application strictly. Its notes explain why: a credit service that quietly turns a bad value into something believable is worse than one that rejects it.

`src/api/audit.py` is the permanent record. Every decision is written down before the reply is sent: time, a fingerprint of the input, score, decision, reasons, model version and cut-off. In consumer credit, as the notes say, this isn't optional. A regulator might check decisions from years ago.

Tested in `tests/test_api.py`, including that the service refuses to start on a mismatched package. `src/api/__init__.py` marks the folder as a package, and `tests/conftest.py` and `tests/__init__.py` hold shared test setup.

> **📁 Files we just created**
> - `src/api/__init__.py`: marks the service folder as a package.
> - `src/api/main.py`: the decision service.
> - `src/api/schemas.py`: strict checks on every application.
> - `src/api/audit.py`: the permanent record of every decision.
> - `tests/test_api.py`: service tests.
> - `tests/conftest.py` and `tests/__init__.py`: shared test setup.

## Step eleven: a demo, and packaging

`app.py` is a demo form, built with Streamlit. You fill in an application and see the decision, the chance of default, the expected loss and the reasons. One honest note: its default values are outside the range the model learned from.

`Dockerfile` packs it all into a box, called a container. Here's a smart part: the model is trained *inside* the box while it's being built.

So the model and its calibration always come from one run. `entrypoint.sh` starts both the service and the demo form inside that box. And `docker-compose.yml` starts it with one command, keeping the audit record outside the box so it survives.

`.github/workflows/ci.yml` checks every change: lint, types, tests, the calibration gate, and a quick check that the box starts. Then it uploads the box to Amazon's container storage.

> **📁 Files we just created**
> - `app.py`: the demo form.
> - `Dockerfile`: packs everything, training the model inside the box.
> - `entrypoint.sh`: starts the service and the demo form.
> - `docker-compose.yml`: starts it all with one command.
> - `.github/workflows/ci.yml`: checks every change, then uploads the box.

## Step twelve: writing it all down

Finally, the documents.

- `DECISIONS.md`: every judgement call and the reasoning behind it, including the counter-arguments.
- `MODEL_CARD.md`: a standard summary of the model: what it's for, how it was tested, and its limits. It's marked "demonstration, not for production lending".
- `RUNBOOK.md`: step-by-step procedures for running the service, written so you can follow them under pressure.
- `CODE_WALKTHROUGH.md`: the project's own walkthrough, in build order, a lot like this one.
- `scripts/render_walkthrough.py`: turns that walkthrough into a PDF you can read on your phone.

> **📁 Files we just created**
> - `DECISIONS.md`: every judgement call, explained.
> - `MODEL_CARD.md`: the model's summary and limits.
> - `RUNBOOK.md`: how to run the service, step by step.
> - `CODE_WALKTHROUGH.md`: the project's own build-order walkthrough.
> - `scripts/render_walkthrough.py`: turns the walkthrough into a phone-friendly PDF.

## So, how's it doing?

On the last 12 months of the made-up book:

- LightGBM made about **385 profit per application**. The scorecard made 13.5% less.
- **The cut-off matters more than the model.** Using 0.5 instead of 0.107 turned a profit into a loss.
- LightGBM's AUC was **0.84**.
- Full fairness between groups would cost **1.64%** of profit.

## What's still missing?

- **The data is made up**, so the numbers won't transfer to a real lender.
- **The real-data path hasn't been run** on the real file.
- **The 75% and 9% are guesses**, and the cut-off moves from 0.05 to 0.21 across sensible values.
- **Age is used by the model and is protected by law**, so it needs legal review.
- **The audit record is a local file**, not special write-once storage.
- **On a fresh install, 4 tests fail**, because a newer version of one statistics tool removed an option the code uses.

## Let's put it all together

So let's look at it in one breath.

We put every **business assumption in one file**. We built a **made-up loan book** where the truth is known for everyone, plus a **real-data path**. We trained **three models**, split **by time**. We made the **chances honest**, with a **gate** that fails the build if they get worse, and **measured fairly** for rare defaults.

Then a **cut-off from money**: 9 ÷ (9 + 75) ≈ 11%. We ran the **hard studies** on declined applicants and **fairness**, and built **reasons** and **monitoring**. We saved every result as **tables and charts**.

Finally, a **package that refuses mismatched parts**, **one decision engine** for live and batch, a **strict decision service** with a **permanent record**, a demo, packaging, and the documents.

Notice how it links. The known truth from step one is what lets the fairness and bias studies be checked at all. The honest chances from step three are what make the money-based cut-off in step four work. And the package guard in step eight protects those honest chances all the way to every live decision.

That's the project. Decide by money, explain every decline, and show the fairness numbers, even the bad ones.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
