# Uplift-Modelling-Decision: Explained to an Engineer

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

---

## Summary

A budget-constrained uplift study on the Hillstrom randomised e-mail trial. Meta-learners (S/T/X, hand-implemented over sklearn/LightGBM bases) and an EconML causal forest are validated on a synthetic DGP with known individual effects, then applied with cross-fitting to Hillstrom. Evaluation uses Qini/AUUC/deciles/transformed outcome with bootstrap CIs and a beats-random test (no AUC). Policy profit is measured from randomised contrasts within the selected slice. Robustness: placebo, balance, cost sensitivity, seed stability. Plus power/MDE and a CUPED demo. The deliverable is `MEMO.md`. ~4,000 lines. One commit, 7 Aug 2026. Results committed in `results/*.json`.

## Architecture

```
src/simulate.py     DGP: randomised W, heterogeneous tau(x), harmed subgroup, noise features
src/naive.py        raw ATE, selection-bias overstatement, response vs uplift targeting on truth
src/data.py         Hillstrom loader (+ mirror), optional Criteo
src/estimators.py   S/T/X-learners (swappable base), CausalForestDML
src/evaluation.py   qini_curve, qini coefficient, AUUC, deciles, transformed outcome, bootstrap, beats-random
src/policy.py       frontier, optimal depth, profit CIs, sleeping-dog accounting
src/experiment.py   power/MDE both directions, cuped_adjust, cuped_demo
src/robustness.py   placebo_test, balance, cost sensitivity, seed stability
src/cli.py          validate | naive | hillstrom | experiment | all [--quick]
src/config.py       $0.10/contact, 30% margin, 30% budget
```

## Key decisions

| Decision | Why |
|---|---|
| Synthetic truth first | Individual effects are unobservable on real data. Estimators must recover known τ. |
| No AUC | It rewards predicting outcome, not uplift |
| Measured, not predicted, policy value | Avoid circular self-evaluation |
| Explicit propensity for known randomisation | Estimating a constant only adds variance |
| Cross-fitting | Out-of-fold uplift scores for evaluation |
| One-sided beats-random test + bootstrap CIs | Is the ranking better than random? |
| Placebo with shuffled W | Does the pipeline find "effects" in noise? |
| Report failures in the memo | The decision depends on them |

## Estimators (from METHODS.md)

- **S-learner:** μ(x, w), τ = μ(x,1) − μ(x,0). Shrinks heterogeneity (SD ratio 0.65 in validation).
- **T-learner:** μ₁, μ₀ fitted separately. Inflates noise (SD ratio 1.69).
- **X-learner:** D₁ = Y − μ̂₀(X) on the treated, D₀ = μ̂₁(X) − Y on the controls. Fit τ̂₁, τ̂₀, combine with `g(x)·τ̂₀ + (1−g(x))·τ̂₁`. PEHE 0.088 vs T 0.106 at a 15/85 split.
- **Causal forest:** EconML `CausalForestDML` (orthogonalised, honest).

Validation (balanced 50/50 synthetic, n=20,000, true ATE 0.0223, 37% sleeping dogs): e.g. S-learner PEHE 0.036, Spearman 0.85, 66% of sleeping dogs detected.

## Evaluation

- **Qini:** `Q(k) = R_t(k) − R_c(k)·N_t(k)/N_c(k)`. The coefficient is the area vs random.
- **AUUC, uplift by decile, transformed-outcome MSE, bootstrap CIs.**
- **Beats-random:** one-sided test on Qini.

**Committed leaderboards (`results/hillstrom.json`):**

| Campaign / outcome | Qini by model (beats random?) |
|---|---|
| Men's conversion | T 10.05 (no, p=0.056), X 8.58 (no), CF 6.43 (no), S 1.84 (no) |
| Men's visit | CF 21.51 (no), S 11.70 (no), X 5.82 (no), T −3.80 (no) |
| Women's conversion | S 13.12 (**yes**), CF 9.07 (**yes**), T 2.49 (no), X −0.25 (no) |

**Policy and robustness (MEMO.md):** 30% budget → $3,066 (CI $782–$5,535) vs $1,674 random. Optimum 85% → $6,241. The men's placebo failed (9.4 vs 2.8 ± 6.3, 2/10 fakes higher). Women's passed. Seed Qini range −0.1 to 6.8, 44% selection overlap. Balance max SMD 0.014. Cost: $0.25/contact → optimum 35%. $0.50 → mailing everyone loses $11,500.

**Experiment:** 4 cells with 10% holdbacks. 0.91% vs 0.68% → 40,900 per cell at α=0.05 (95% confidence), 80% power. CUPED: −50% variance on visits, −1.7% on conversion.

**Tests:** 136 pass in my run (~44 s), all on synthetic data (the README says 135).

## Known weaknesses

- **The men's targeting gain isn't statistically established** (placebo fail, no estimator beats random on men's conversion).
- **Few positives:** ~289 incremental conversions.
- **Old, short-window data** (2008, two weeks).
- **Assumed economics** drive the depth.
- **Unstable rankings** below the top decile.
- **The memo headline recommends model targeting at 30%** even though the leaderboard's beats-random test fails for men's conversion. The memo caveats this, but a reader skimming the first paragraph may miss it.
