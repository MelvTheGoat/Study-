# Credit-Risk-Decisioning: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

---

## 1. Credit risk fundamentals

**Simple explanation:** expected loss = probability of default × loss given default × exposure at default.

**In this project:** PD from the models, LGD 0.75 assumed, EAD = loan amount, and profit and loss accounting per decision.

**Also be ready to explain:** PD/LGD/EAD definitions, default definitions (90 days past due at 12 months), vintage analysis, roll rates, Basel IRB basics, IFRS 9 staging (12-month vs lifetime ECL).

---

## 2. Scorecards (WOE / IV)

**Simple explanation:** bin each variable, convert bins to weight of evidence, fit a logistic regression, and turn coefficients into points.

**In this project:** monotonic WOE binning, an information-value table, a points table, points-shortfall reason codes.

**Also be ready to explain:** WOE = ln(%good/%bad), IV thresholds, why monotonicity matters, points-to-double-odds scaling, reject reasons.

---

## 3. Cost-sensitive decision making

**Simple explanation:** choose the cutoff that maximises money, not a statistical score.

**In this project:** the closed-form `p* = C_fp/(C_fp+C_fn)`, an empirical sweep, a sensitivity grid, and comparison with the F1 and 0.5 cutoffs.

**Also be ready to explain:** why the closed form needs calibration, profit curves, approval rate vs bad rate trade-offs, and risk-based pricing (instead of a hard decline).

---

## 4. Probability calibration

**Simple explanation:** make "10%" mean 10%.

**In this project:** raw/Platt/isotonic, Brier, ECE, MCE, calibration slope and intercept, in-period vs out-of-time.

**Also be ready to explain:** reliability diagrams, why isotonic overfits small calibration sets, recalibration under drift, and the effect of class weighting on calibration.

---

## 5. Model evaluation for imbalanced problems

**Simple explanation:** accuracy lies when most people don't default.

**In this project:** ROC-AUC, Gini, PR-AUC vs base rate, KS, precision@capacity, and "nobody defaults" accuracy as a baseline.

**Also be ready to explain:** Gini = 2·AUC − 1, KS statistic, lift and gains charts, and out-of-time vs out-of-sample validation.

---

## 6. Reject inference and selection bias

**Simple explanation:** you only see outcomes for people you approved, so your data is biased.

**In this project:** fuzzy augmentation, parcelling, Heckman two-step, oracle, and MAR vs MNAR.

**Also be ready to explain:** MCAR/MAR/MNAR, exclusion restrictions, the inverse Mills ratio, and using a small random-approval "test cell" to get unbiased data.

---

## 7. Algorithmic fairness

**Simple explanation:** check whether groups are treated differently, and why.

**In this project:** DPD, AIR (the four-fifths rule), EOD, per-group calibration, the λ trade-off, and separating true risk gaps from model-made gaps.

**Also be ready to explain:** disparate treatment vs disparate impact, proxy discrimination, the impossibility results (you can't equalise everything with different base rates), fairness through unawareness, and ECOA/Reg B adverse-action rules (US) or local equivalents.

---

## 8. Explainability

**Simple explanation:** say *why* a decision was made.

**In this project:** points-shortfall reasons (deployed), SHAP, and protected-basis flags.

**Also be ready to explain:** SHAP values (Shapley additivity), TreeSHAP, background data dependence, and reason-code stability.

---

## 9. Model monitoring

**Simple explanation:** watch whether inputs and outcomes change after launch.

**In this project:** score and feature PSI, vintage curves, combined-watch triggers.

**Also be ready to explain:** PSI formula and thresholds (0.1/0.25), CSI, concept vs data drift, outcome lag, and champion/challenger.

---

## 10. MLOps and model governance

**Simple explanation:** ship models safely and traceably.

**In this project:** versioned bundles with hashes and a shared run ID, refuse-on-mismatch, one engine for API and batch, an append-only audit, a calibration regression gate, a model card, decisions log and runbook, a Docker multi-stage build, and push to AWS ECR.

**Also be ready to explain:** SR 11-7 style model risk management, model cards, reproducibility, WORM storage, and dependency pinning (see the statsmodels break).
