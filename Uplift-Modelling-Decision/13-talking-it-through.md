# Who to Target with Marketing: Let's Talk It Through

*No computer, no slides. Just you and me, talking through how this project was built, from the very first step to the last. As we go, I'll name every file we create and why we need it. Look for the 📁 boxes: they list the files made in each step. Now and then I'll show you a few lines of the real code, but you don't need them to follow along.*

*One thing up front: this is a study, not a running app. It's a careful, repeatable analysis that ends in a short decision memo.*

---

## Okay, so what are we building?

Alright. A marketing director comes to us with a question. "We have a fixed budget. We can only e-mail some of our customers. Who should get the campaign?"

Most people would say "the customers most likely to buy". But some of those would buy anyway, e-mail or not. E-mailing them wastes money.

And some customers are even put *off* by being contacted. They're called "sleeping dogs", as in "let sleeping dogs lie".

So the real question is: who buys *because* we contacted them? That extra chance of buying, caused by the e-mail, is called **uplift**.

So here's our project: find the customers with the highest uplift, work out what that's worth in money, and then try hard to prove ourselves wrong. And it ends in a two-page memo for the director.

## So what do we need?

1. **Settings**, with every business assumption in one place.
2. **Made-up data where the true uplift is known**, to test our methods.
3. **The common mistake, shown clearly**, so we can explain why it misleads.
4. **Real data from a fair experiment.**
5. **Uplift methods**, all with one shape.
6. **The right way to measure them**, which is *not* the usual way.
7. **A way to turn scores into a decision**, under a budget.
8. **Tests that try to break the result.**
9. **A plan for the next experiment.**
10. **A memo** that tells the truth, including the bad news.

## Step zero: set up the workshop

First, a `README.md`, the front page. A `.gitignore`. `requirements.txt` and `pyproject.toml` list the tools and project details. And `SETUP.md` explains how to start the project fresh, with no previous history.

Two empty folders are kept with `.gitkeep` files: `data/` for the downloaded data, and `results/` for everything the study produces.

The code lives in `src/`. Its `__init__.py` says what it is: "who should receive the campaign under a fixed budget". `src/arrays.py` is tiny: one shared type label used for every number array in the project.

Then `src/config.py`. Every number a business reader might argue with lives here: $0.10 per e-mail, a 30% profit margin, and a budget covering 30% of customers. Change one, re-run, and the whole study updates.

> **📁 Files we just created**
> - `README.md`: the project's front page.
> - `.gitignore`: files git should not save.
> - `requirements.txt` and `pyproject.toml`: tools and project details.
> - `SETUP.md`: how to start the project fresh.
> - `data/.gitkeep` and `results/.gitkeep`: keep the empty data and results folders.
> - `src/__init__.py`: marks the code folder as a package.
> - `src/arrays.py`: one shared type label for number arrays.
> - `src/config.py`: every business assumption in one place.

## Step one: data where we know the answer

Here's the hard part about uplift. You can never see the same person both e-mailed *and* not e-mailed. So in real data, you never know anyone's true uplift.

So how do you know if a method works? You test it first where you *do* know the answer. `src/simulate.py` creates made-up customers where each person's true uplift is known. Its notes say it exists so every method can be checked against an answer known in advance.

It builds in a mix: some people respond a lot, some barely respond, and a group of real sleeping dogs who are put off by contact. It also adds useless facts, to see if methods get fooled by noise.

Tested in `tests/test_simulate.py`. As its notes say, the simulator is the yardstick for everything else, "so it is tested hardest".

> **📁 Files we just created**
> - `src/simulate.py`: made-up customers where the true uplift is known.
> - `tests/test_simulate.py`: the hardest-tested file, because everything relies on it.

## Step two: the common mistake

Before doing it right, we show how it usually goes wrong. `src/naive.py` calls itself "the wrong answer, computed carefully".

Almost every campaign report does this: "people we contacted bought more than people we didn't, so the campaign worked." But if the contact list wasn't chosen at random, that comparison is badly biased. Here, it made the effect look **7.5 times** bigger than the truth.

It also shows that picking "people most likely to buy" is not the same as picking "people most likely to be persuaded". Tested in `tests/test_naive.py`.

> **📁 Files we just created**
> - `src/naive.py`: the common mistake, shown clearly.
> - `tests/test_naive.py`: tests for that demonstration.

## Step three: real data from a fair experiment

Now the real data. `src/data.py` loads the Hillstrom e-mail challenge. That's 64,000 customers from 2008, split at random three ways: a men's e-mail, a women's e-mail, and no e-mail.

Why does random matter so much? Because chance decided who got the e-mail. So the groups are alike in every other way, and any difference in buying was caused by the e-mail. That's what makes a fair comparison possible.

The original download link is often blocked, so it falls back to an identical copy online. There's also an optional, much bigger dataset from an advertising company, Criteo.

> **📁 Files we just created**
> - `src/data.py`: loads the real 64,000-customer e-mail trial.

## Step four: the uplift methods

`src/estimators.py` holds four methods, all with one simple shape: learn, then predict.

Three are "meta-learners": standard recipes that build uplift from ordinary prediction models. The **S-learner** uses one model, with "was e-mailed" as one input. The **T-learner** uses two separate models, one per group. The **X-learner** is smarter, using each group's model to fill in the other's missing outcome.

These three are written out by hand, so you can see exactly how they work. The fourth is the **causal forest**, from a specialised library called EconML.

Tested in `tests/test_estimators.py`, against the known truth. For example, it checks the S-learner tends to shrink differences between people, and the T-learner tends to exaggerate them.

> **📁 Files we just created**
> - `src/estimators.py`: four uplift methods, with one shape.
> - `tests/test_estimators.py`: checks each method against the known truth.

## Step five: measuring the right way

`src/evaluation.py`, and its notes start with something surprising: it deliberately contains **no AUC**, the usual score.

Why? Because AUC measures how well you rank people by *whether they bought*. But we don't want buyers. We want *persuadable* people.

A model could score brilliantly on AUC and be useless for targeting.

So instead, it uses the Qini curve. That shows how many extra purchases you get as you contact more of the top-ranked customers. It also breaks results into 10 groups, adds honest error ranges, and asks a simple question: is this better than picking at random?

Tested in `tests/test_evaluation.py`. Its key checks are the obvious ones: a random score must give a Qini of about zero. If it didn't, the measure itself would be broken.

Let's pause. We have made-up data where we know the truth, a demonstration of the common mistake, real data from a fair experiment, four methods, and the right way to measure them. Now we turn scores into a decision.

> **📁 Files we just created**
> - `src/evaluation.py`: the Qini curve and honest error ranges, with no AUC.
> - `tests/test_evaluation.py`: checks the measures on cases with known answers.

## Step six: from scores to a decision

`src/policy.py`, and its notes make a sharp point: a ranking is not a policy. The model gives an order. The decision is where to cut it.

So for every budget size, it works out the profit. And here's the honest part: it *measures* the extra sales from the real trial, by comparing e-mailed and not e-mailed people inside each chosen group. It never trusts the model's own guesses for this. Otherwise the model would be marking its own homework.

Tested in `tests/test_policy.py`, using cases you can check by hand. As its notes say, a profit number that's wrong in a way nobody notices is the most expensive kind of mistake.

> **📁 Files we just created**
> - `src/policy.py`: profit at each budget size, measured from the real trial.
> - `tests/test_policy.py`: hand-checkable profit tests.

## Step seven: trying to break it

`src/robustness.py` holds "checks designed to make the model fail". Each one has a right answer that doesn't depend on the model being any good.

The **placebo test** shuffles who got the e-mail and re-runs everything. Any "effect" found in shuffled data must be pure luck.

The **balance check** confirms the groups really were alike before the e-mail. **Cost sensitivity** re-runs with different e-mail costs. And **seed stability** checks whether results change with a different random start.

Tested in `tests/test_robustness.py`. Its notes put it perfectly: a placebo test that never fails is worse than no placebo test. So the tests check that these checks actually catch problems.

> **📁 Files we just created**
> - `src/robustness.py`: the checks that try to make the model fail.
> - `tests/test_robustness.py`: checks that those checks really work.

## Step eight: planning the next experiment

`src/experiment.py`, and its notes say it well: a model fitted to a 2008 mailing is a hypothesis, not a result. So this file sizes the experiment that would confirm it or kill it.

It works out how many customers a follow-up test needs to detect a real difference. It also shows CUPED, a trick that uses people's past behaviour to cut down noise. Tested in `tests/test_experiment.py`.

> **📁 Files we just created**
> - `src/experiment.py`: sizes the follow-up test, and shows CUPED.
> - `tests/test_experiment.py`: checks the sizing and CUPED.

Let's circle back. We have methods checked against the truth, the right measures, a money-based decision, tests that try to break it, and a plan for the next experiment. Now we run it all and write it up.

## Step nine: running the whole study

`src/cli.py` runs it all, in three stages, in a deliberate order. First, **validate**: every method against the known truth. If a method can't find the answer there, it has no business being used on real data.

Then **naive**: the common mistake. Then **hillstrom**: the real study. Plus **experiment**, to plan the follow-up.

Every number in the memo traces back to a file it writes in `results/`:

- `validation.json`, `validation_balanced_50_50.csv`, `validation_imbalanced_15_85.csv`: how well each method found the known truth, with equal and unequal group sizes.
- `naive.json`: the common mistake's numbers.
- `hillstrom.json`: the real study's main results.
- `models_mens_conversion.csv`, `models_mens_visit.csv`, `models_womens_conversion.csv`: each method's scores, for each campaign and outcome.
- 12 `deciles_` files, one per campaign and method (listed just below): results split into 10 groups.
- `frontier_mens_conversion.csv`, `frontier_mens_visit.csv`, `frontier_womens_conversion.csv`: profit at every budget size.
- `balance_mens_conversion.csv`, `balance_mens_visit.csv`, `balance_womens_conversion.csv`: the balance checks.
- `cost_sensitivity_mens_conversion.csv`, `cost_sensitivity_mens_visit.csv`, `cost_sensitivity_womens_conversion.csv`: results at different e-mail costs.
- `experiment.json`: the follow-up test plan.

The 12 decile files are `deciles_mens_conversion_causal-forest.csv`, `deciles_mens_conversion_s-learner.csv`, `deciles_mens_conversion_t-learner.csv`, `deciles_mens_conversion_x-learner.csv`, `deciles_mens_visit_causal-forest.csv`, `deciles_mens_visit_s-learner.csv`, `deciles_mens_visit_t-learner.csv`, `deciles_mens_visit_x-learner.csv`, `deciles_womens_conversion_causal-forest.csv`, `deciles_womens_conversion_s-learner.csv`, `deciles_womens_conversion_t-learner.csv` and `deciles_womens_conversion_x-learner.csv`.

`.github/workflows/ci.yml` checks every change. And `tests/conftest.py` holds shared test setup, all on made-up data, so the tests never need the internet.

> **📁 Files we just created**
> - `src/cli.py`: runs the whole study in a deliberate order.
> - `results/`: every result file the memo quotes.
> - `.github/workflows/ci.yml`: checks every change.
> - `tests/conftest.py`: shared test setup with made-up data.

## Step ten: writing it up

- `MEMO.md`: the deliverable. Two pages for the marketing director: the decision, the uncertainty, and what would change it.
- `METHODS.md`: the technical companion. What each method assumes, how each measure works, and where each one fails.
- `CODE_WALKTHROUGH.md`: the project's own build-order walkthrough, starting from an empty folder. It was later removed, and `METHODS.md` now does that job.

> **📁 Files we just created**
> - `MEMO.md`: the two-page decision memo.
> - `METHODS.md`: the technical companion.
> - `CODE_WALKTHROUGH.md`: the project's own walkthrough (later removed).

## Step eleven: a website anyone can use

A memo only reaches people who read memos. So, in October, we add a website. It explains the study in plain English, and it has a tool: **upload your own campaign as a CSV file and get the whole analysis on it.**

Here's the nice part: there's no server. Everything runs inside your browser, so your file is never sent anywhere. It's plain HTML, CSS and JavaScript, with no build step and nothing to install.

`web/index.html` is the one page that loads everything, and `web/styles.css` makes it look right. In `web/js/`, `main.js` switches between pages, `charts.js` draws the charts, and `csv.js` reads your file. `stats.js`, `format.js`, `dom.js` and `copy.js` are small helpers for maths, number formatting, building the page, and the words on it. Each page has its own file in `web/js/pages/`: the overview, did it work, targeting, money, sleeping dogs, can we trust it, planning a test, about, and the analyse tool.

The site's numbers come from one file, `web/data.json`. `scripts/build_web_data.py` packs everything in `results/` into it. And `web/sample-campaign.csv` is a built-in example, so you can see the tool work before using your own data.

Now, a danger. `web/js/analysis.js` redoes the Python maths in JavaScript. Two copies of the same formula will drift apart one day.

So `tests/test_web_analysis_parity.py` makes up a campaign, runs both versions, and fails if any number disagrees. `tests/test_web_parity.py` does the same for the sample-size calculator.

Last, `.github/workflows/pages.yml` puts the site online with GitHub Pages, and `DEPLOY.md` explains how. The workflow rebuilds `web/data.json` and refuses to publish if it doesn't match the saved one. That stops the site quietly showing old numbers.

> **📁 Files we just created**
> - `web/index.html`, `web/styles.css`: the page and its looks.
> - `web/js/main.js`, `charts.js`, `csv.js`, `stats.js`, `format.js`, `dom.js`, `copy.js`: page switching, charts, reading files, and helpers.
> - `web/js/pages/`: one file per page, including `analyse.js`, the upload tool.
> - `web/js/analysis.js`: the study's maths, redone in JavaScript.
> - `web/data.json`: every result, packed for the site.
> - `web/sample-campaign.csv`: a built-in example campaign.
> - `scripts/build_web_data.py`: packs `results/` into `web/data.json`.
> - `tests/test_web_parity.py`, `tests/test_web_analysis_parity.py`: check the JavaScript matches the Python.
> - `.github/workflows/pages.yml`: publishes the site.
> - `DEPLOY.md`: how to put the site online.

## So, how's it doing?

For the men's e-mail campaign:

- The e-mail clearly worked overall: 1.25% of e-mailed men bought, against 0.57% with no e-mail.
- At a 30% budget, targeting made about **$3,066**, against **$1,674** at random. But the honest range was wide: $782 to $5,535.
- **A bigger budget mattered more than smarter targeting**: about $3,170 against about $1,390.

And the warnings:

- For men's sales, **no method clearly beat random**. For women's sales, two did.
- The **placebo test failed** for the men's campaign.
- **Sleeping dogs weren't confirmed.** The group flagged as "harmed" actually bought more.

## What's still missing?

- **Only about 289 extra sales** happened in the whole trial, which is very little to learn from.
- **The data is from 2008**, and only covers two weeks.
- **Rankings change between runs**: only 44% of chosen customers stayed the same.
- **The costs drive the answer.** At $0.50 per e-mail, e-mailing everyone loses $11,500.
- **It's a study, not a live system.**

## Let's put it all together

So let's look at it in one breath.

We put every **business assumption in one file**. We built **made-up customers with a known uplift**, and showed the **common mistake** that inflates results 7.5 times. We loaded a **real, fair experiment** of 64,000 customers. We built **four uplift methods** and checked them against the truth.

We **measured the right way**, with no AUC. We turned scores into a **money decision**, measured from the trial, not the model. We **tried hard to break it**, and planned the **next experiment**. Then we ran it all in **one ordered study** and wrote an **honest memo**.

Notice how it links. The known truth from step one is what lets us trust the methods in step four. The fair experiment from step three is what lets the decision in step six be *measured*, not guessed. And the placebo test in step seven is what stops us over-claiming in the memo.

That's the project. Find who's persuadable, measure honestly, and report the bad news too.

## Where to go next

- For the whole project in short, read `00-start-here.md`.
- For the system with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word, read `11-technical-terms.md`.
- For every tool, read `12-tools-and-why.md`.
- For the full technical detail, read `01-system-design.md` and `06-explain-to-technical.md`.
