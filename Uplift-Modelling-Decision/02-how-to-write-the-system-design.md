# Uplift-Modelling-Decision: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

This is an analysis design (a decision study), so the whiteboard is about **data → estimation → evaluation → decision → validation**, not servers.

---

## Step 1: Requirements (2 min)

One line:
> "Given a fixed budget, choose which customers to contact to maximise *incremental* profit, and prove the choice is better than random."

**Functional**
1. Estimate each customer's uplift (the change in conversion if contacted).
2. Rank customers and choose how deep to go under the budget.
3. Measure the profit of that policy honestly.
4. Test whether the targeting beats random.
5. Design a validation experiment.

**Non-functional**
- **Causal validity:** use randomised data, and never self-grade with predictions.
- **Reproducible:** every memo number traces to `results/`.
- **Honest uncertainty:** CIs, placebo tests, seed stability.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Hillstrom file | 64,000 (42,613 in the two arms analysed) |
| Conversion | treated 1.25%, control 0.57% |
| Incremental conversions available | ~289 |
| Contact cost / margin | $0.10 / 30% |
| Budget | 30% of the file (12,784 contacts) |

**Say:** "Very few positive signals. The data, not the model, limits what can be learned."

---

## Step 3: High-level boxes (2 min)

```
[Synthetic data with known effect] -> [S/T/X-learner, causal forest] -> [Recovery metrics]   (validate first)
[Hillstrom randomised trial] -> [Cross-fitted uplift scores] -> [Qini/AUUC/deciles + bootstrap + beats-random]
      -> [Policy: frontier, measured profit per depth] -> [Stress tests: placebo, balance, cost, seeds]
      -> [Experiment design: power/MDE, CUPED] -> [MEMO.md]
```

---

## Step 4: Deep dive (10 min)

### 4a. Why uplift, not response
- A response model ranks "who buys". Uplift ranks "who buys *because* of contact".
- On known truth: the response model's top 20% captured 440 incremental sales, uplift 657, and the response model mailed 5× more harmed customers.

### 4b. Estimators
- **S-learner:** one model with treatment as a feature. It shrinks effects (0.65× the true spread).
- **T-learner:** separate models per arm. It inflates noise (1.69×).
- **X-learner:** impute counterfactuals, then combine by propensity. Better under imbalance.
- **Causal forest (DML):** orthogonalised, with an honest forest.

### 4c. Evaluation (no AUC)
- **Qini curve:** `Q(k) = treated_responders(k) − control_responders(k)·n_t(k)/n_c(k)`.
- Qini coefficient, AUUC, uplift by decile, transformed outcome, bootstrap CI, one-sided beats-random.

### 4d. Policy
- For each depth, measure incremental effect **from treated vs control inside the slice**.
- Profit = margin × incremental revenue − cost × contacts.
- Frontier: 30% → $3,066. Optimum at 85% → $6,241.

### 4e. Stress tests
- Placebo: shuffle the treatment 10×. The men's campaign failed (real 9.4 vs 2.8 ± 6.3).
- Balance (max SMD 0.014), cost sweep, seed stability (44% overlap).

### 4f. Validation experiment
- 4 cells (model vs business-as-usual, each with a 10% holdback), ~40,900 per cell.
- CUPED helps continuous metrics (−50% variance) and not 1%-rate conversion (−1.7%).

---

## Step 5: Bottlenecks and risks (2 min)

1. **Few positives** (~289 incremental conversions).
2. **Assumed economics** swing the optimal depth (85% → 35% at $0.25/contact).
3. **Unstable rankings** below the top decile.
4. **A short outcome window** hides long-term harm.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Uplift metrics (Qini) | AUC | AUC rewards the wrong thing | Less familiar |
| Measured profit | Predicted profit | Avoids circularity | Noisier |
| Synthetic validation first | Trust real-data fit | Real individual effects are unobservable | Extra work |
| Report the placebo failure | Hide it | Honesty. It changes the recommendation. | Weaker headline |
| Recommend testing | Roll out | Targeting gain not proven | Slower |
