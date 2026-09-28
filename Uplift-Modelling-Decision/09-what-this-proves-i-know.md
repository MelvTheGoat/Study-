# Uplift-Modelling-Decision: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

---

## 1. Causal inference basics

**Simple explanation:** separate what an action *causes* from what merely goes along with it.

**In this project:** randomised treatment, the average treatment effect (ATE), the conditional effect (CATE), and selection bias shown with a 7.5× overstatement.

**Also be ready to explain:** potential outcomes, the fundamental problem of causal inference, confounding, randomisation, unconfoundedness, SUTVA and overlap.

---

## 2. Uplift modelling / heterogeneous treatment effects

**Simple explanation:** estimate how much each person's outcome changes if treated.

**In this project:** S/T/X-learners and a causal forest, sleeping dogs.

**Also be ready to explain:** the four customer types (persuadables, sure things, lost causes, sleeping dogs), meta-learners vs direct methods, DML (double machine learning), and honest forests.

---

## 3. Uplift evaluation metrics

**Simple explanation:** measure whether ranking by predicted uplift actually captures more incremental outcomes.

**In this project:** Qini curve and coefficient, AUUC, uplift by decile, transformed outcome, a beats-random test.

**Also be ready to explain:** why AUC is wrong here, the Qini vs uplift curve, transformed-outcome regression (Y* = Y·(W−e)/(e(1−e))), and PEHE on synthetic data.

---

## 4. Decision-making under a budget

**Simple explanation:** choose how many and whom to treat to maximise profit.

**In this project:** a budget frontier, the profit-maximising depth, profit per 1,000 contacts, profit CIs.

**Also be ready to explain:** marginal vs average return, the knapsack framing, cost sensitivity, and why efficiency and total profit point different ways.

---

## 5. Experiment design and power

**Simple explanation:** plan a test big enough to detect the effect you care about.

**In this project:** MDE and sample size in both directions, 4 cells with holdbacks, pre-set success criteria.

**Also be ready to explain:** alpha, power, MDE, two-proportion z-tests, multiple testing, peeking, and sequential tests.

---

## 6. Variance reduction (CUPED)

**Simple explanation:** use pre-experiment data to remove predictable noise and get sharper results.

**In this project:** −50% variance on visits, −1.7% on conversion.

**Also be ready to explain:** the CUPED formula (Y − θ(X − mean X)), why it depends on the pre/post correlation, and CUPAC.

---

## 7. Robustness and falsification

**Simple explanation:** try to break your own result.

**In this project:** a placebo test with shuffled labels, covariate balance (SMD), cost sensitivity, seed stability.

**Also be ready to explain:** permutation tests, standardised mean difference, and why seed instability matters for deployment.

---

## 8. Bootstrap and uncertainty

**In this project:** bootstrap CIs for Qini, decile lifts and profit.

**Also be ready to explain:** percentile vs BCa intervals, and why intervals matter more than point estimates for decisions.

---

## 9. Communicating analysis to decision-makers

**Simple explanation:** turn statistics into a short, honest recommendation.

**In this project:** a two-page memo with a recommendation, the key uncertainty, the next experiment, and what would change the decision.

**Also be ready to explain:** how to present negative results, and how to separate "statistically solid" from "promising".

---

## 10. Reproducible research engineering

**In this project:** a CLI pipeline, results as JSON/CSV, strict mypy, 135 tests on synthetic data, CI on Python 3.10–3.12.

**Also be ready to explain:** testing statistical code against known truth, and seeding.
