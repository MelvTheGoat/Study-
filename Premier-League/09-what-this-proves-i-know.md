# Premier-League: What This Proves I Know

Repo: https://github.com/MelvTheGoat/Premier-League

---

## 1. Feature engineering for time series / sports

**Simple explanation:** turn raw history into numbers that describe a match's situation.

**In this project:** rolling windows (4/6/10), table gaps, congestion, manager tenure, H2H, venue splits, season shape, home/away/difference columns.

**Also be ready to explain:** window choice, exponential vs simple moving averages, target encoding risks, and why difference features help tree models.

---

## 2. Leakage and point-in-time correctness

**Simple explanation:** only use information that existed when the prediction would have been made.

**In this project:** a forward-only state machine, the gameweek-level cutoff, walk-forward backtesting.

**Also be ready to explain:** look-ahead bias, walk-forward vs k-fold CV, training/serving skew, and why random splits break time series.

---

## 3. Rating systems (Elo)

**Simple explanation:** each team has a number. After a match, the winner takes points from the loser, more for an upset.

**In this project:** K=20, home advantage 60, regression between seasons, division offsets, cross-competition.

**Also be ready to explain:** the expected score formula `1/(1+10^(−diff/400))`, choosing K, Glicko (rating uncertainty), and margin-of-victory adjustments.

---

## 4. Gradient boosting (LightGBM)

**Simple explanation:** add many small decision trees, each fixing the errors of the last.

**In this project:** regularised settings, early stopping on a chronological tail, log-loss objective.

**Also be ready to explain:** learning rate vs rounds, leaves and min child samples, row/column subsampling, L2, feature importance (gain vs split), and missing-value handling.

---

## 5. Multinomial logistic regression and ensembling

**Simple explanation:** a linear model that outputs probabilities for 3+ classes via softmax. Blending averages two models' probabilities.

**In this project:** a compact core feature set, a 0.6/0.4 blend, renormalised.

**Also be ready to explain:** softmax, regularisation, why blending models that fail differently helps, stacking vs blending.

---

## 6. Probability calibration and proper scoring rules

**Simple explanation:** predicted probabilities should match how often things happen.

**In this project:** log loss, Brier score, a reliability table.

**Also be ready to explain:** reliability diagrams, Platt scaling vs isotonic regression, why accuracy is a poor metric for probabilistic forecasts, and expected calibration error.

---

## 7. Poisson goal models (Dixon-Coles)

**Simple explanation:** model each team's goal rate from attack and defence strengths, then get score probabilities.

**In this project:** attack × defence × home advantage, rho low-score correction, time decay, ridge, Championship offset, the scoreline consistent with the outcome.

**Also be ready to explain:** Poisson assumptions, the bivariate Poisson, why independence fails at low scores, maximum likelihood fitting, and expected goals (xG).

---

## 8. Cold-start problems

**Simple explanation:** how to predict for something with little history.

**In this project:** promoted clubs, handled with cross-division Elo and Championship-fitted Dixon-Coles.

**Also be ready to explain:** priors and shrinkage, transfer across domains, and cold start in recommender systems.

---

## 9. Handling missing and lagging data

**Simple explanation:** don't let the model depend on data it won't have at prediction time.

**In this project:** a coverage gate per run, rating age as a feature, missing availability treated as unknown.

**Also be ready to explain:** missing-at-random vs not, indicator features, and imputation trade-offs.

---

## 10. MLOps: scheduled retraining and publishing

**Simple explanation:** automate retrain → predict → publish → verify.

**In this project:** a daily GitHub Action with change detection, render checks, fast-forward branch sync, live-site checks, append-only runs, a committed read-only DB on Vercel.

**Also be ready to explain:** model registries, data versioning, monitoring and alerting, the "green CI, broken prod" failure, and serverless limits.
