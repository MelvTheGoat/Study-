# Uplift-Modelling-Decision: System Design

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

> This is a **causal-inference study** that produces a decision memo (`MEMO.md`), not a deployed service. The "system" is a reproducible analysis pipeline. The repo arrived as **one commit** (7 Aug 2026). Results are committed in `results/*.json`.

## The problem, in 3 lines

With a fixed marketing budget, who should get the campaign?
Not "who is likely to buy", but "who buys **because** we contacted them". Those are different people, and some customers are even put off by contact ("sleeping dogs").
This study uses a real randomised e-mail trial (Hillstrom, 64,000 customers) plus synthetic data with a known true effect, to pick targets, price the budget decision, and check whether the targeting gain is real.

## Diagram

```mermaid
flowchart TB
    subgraph DATA["Data"]
        SIM[simulate.py<br/>known individual effect,<br/>heterogeneous, harmed subgroup,<br/>noise features]
        HILL[Hillstrom 2008<br/>64k customers, 3 arms,<br/>mirror fallback]
        CRI[Criteo-UPLIFT<br/>optional]
    end

    subgraph VALID["1. Validate on known truth"]
        EST[S / T / X-learners<br/>+ causal forest (EconML)]
        REC[Recovery: PEHE, ATE bias,<br/>rank correlation, SD ratio,<br/>sleeping dogs found]
    end

    NAIVE[2. naive.py<br/>treated vs control gap,<br/>selection-bias demo (7.5x),<br/>response model vs uplift model]

    subgraph REAL["3. Real study (Hillstrom)"]
        CV[Cross-fitted uplift scores]
        EVAL[Qini, AUUC, deciles,<br/>transformed outcome,<br/>bootstrap CIs, beats-random test<br/>NO AUC]
        POL[Policy: budget frontier,<br/>profit with CIs, sleeping-dog accounting]
        ROB[Stress tests: placebo, balance,<br/>cost sensitivity, seed stability]
    end

    EXP[4. experiment.py<br/>power / MDE, CUPED demo]
    MEMO[MEMO.md<br/>2-page decision memo]

    SIM --> EST --> REC
    SIM --> NAIVE
    HILL --> CV --> EVAL --> POL --> ROB --> MEMO
    EST -.same estimators.-> CV
    EXP --> MEMO
    NAIVE --> MEMO
    CRI -.-> CV
```

## Each part, and why it's there

| Part | Code | What it does | Why it's there |
|---|---|---|---|
| Simulator | `src/simulate.py` | Randomised data with a known individual treatment effect, heterogeneity, a genuinely harmed subgroup ("sleeping dogs") and inert noise features. | On real data the individual effect is never observed. Estimators must prove themselves on known truth first. |
| Naive analysis | `src/naive.py` | The treated-vs-control gap, done carefully. A simulated **non-randomised** list overstates the true effect **7.5×**. A response model vs an uplift model on known truth. | Shows why the obvious analysis misleads. |
| Data | `src/data.py` | Hillstrom loader with a byte-identical GitHub mirror fallback. Optional Criteo. | The canonical URL is plain HTTP and often blocked. |
| Estimators | `src/estimators.py` | S-, T-, X-learners by hand over swappable sklearn/LightGBM bases, plus EconML `CausalForestDML`. | Standard meta-learners with known failure modes. |
| Evaluation | `src/evaluation.py` | Qini curve and coefficient, AUUC, uplift by decile, transformed-outcome MSE, bootstrap intervals, a one-sided beats-random test. **No AUC.** | AUC rewards predicting buyers, which is the wrong target. |
| Policy | `src/policy.py` | Budget frontier (profit by share mailed), optimal depth, profit CIs, sleeping-dog accounting. **Effects are measured from the randomised holdout inside the selected group**, never from model predictions. | Avoids circular self-grading. |
| Experiment design | `src/experiment.py` | Power/MDE in both directions, and a CUPED variance-reduction demo. | Designs the follow-up test. |
| Stress tests | `src/robustness.py` | Placebo test (shuffled treatment labels), covariate balance, cost sensitivity, seed stability. | Tries to break the finding. |
| CLI | `src/cli.py` | `validate`, `naive`, `hillstrom`, `experiment`, `all` (`--quick`). Writes `results/`. | Every memo number traces to a file. |
| Config | `src/config.py` | $0.10 per contact, 30% gross margin, a budget of 30% of the file. | The business assumptions, in one place. |
| Memo | `MEMO.md` | A 2-page plain-language decision memo for a marketing director. | The deliverable. |

## Tech stack

| Tool | What it's used for | Why this one |
|---|---|---|
| Python 3.10–3.12 | Everything | Standard |
| numpy (≥2.0), pandas, scipy | Data and stats | `np.trapezoid` needs NumPy 2 |
| scikit-learn, LightGBM | Base learners | Swappable bases for the meta-learners |
| EconML | Causal forest (DML) | A well-tested implementation |
| pytest, ruff, mypy (strict) | Quality | 136 tests pass (my run), all synthetic |
| GitHub Actions | CI on 3.10–3.12 | Lint, types, tests |

## Data flow, step by step

1. **Validate:** simulate data with a known effect, fit S/T/X and the causal forest, and measure recovery (PEHE, ATE bias, correlations, SD ratio, sleeping dogs detected).
2. **Naive:** compute the raw lift, then show the selection-bias overstatement and the response-vs-uplift targeting gap on known truth.
3. **Hillstrom:** for each campaign/outcome (men's conversion, men's visit, women's conversion), cross-fit uplift scores, build the Qini, deciles and bootstrap CIs, and run the beats-random test.
4. **Policy:** rank by predicted uplift and, for each depth, **measure** incremental conversions and spend from treated vs control inside that slice → profit = margin × incremental revenue − $0.10 × contacts.
5. **Stress tests:** placebo (10 shuffled-label refits), balance, cost sweep, seed stability.
6. **Experiment:** size a 4-cell validation test, and demonstrate CUPED.
7. **Memo:** summarise the decision, the uncertainty and what would change it.

## Trade-offs and limits

**Key results (Hillstrom, from MEMO.md and `results/hillstrom.json`):**
- Men's e-mail conversion 1.25% vs control 0.57% (a real, randomised lift).
- At a 30% budget: **$3,066 profit** (95% CI $782–$5,535) with model targeting vs **$1,674** random. The profit-maximising depth is 85% of the file ($6,241).
- **Raising the budget is worth ~$3,170. Better targeting ~$1,390.**
- Top decile lift 1.36 pp (CI 0.71–2.02) vs a 0.68 average. **Deciles 2–10 show no trend.**
- **Placebo test failed on the men's campaign** (real Qini 9.4 vs shuffled mean 2.8, SD 6.3, 2 of 10 fake runs scored higher). Women's passed.
- In the committed leaderboard for men's conversion, **no estimator passes the beats-random test** (best one-sided p = 0.056). For women's conversion, the S-learner and causal forest do.
- The model's "harmed" group on men's actually **gained** +0.44 pp. Sleeping dogs weren't confirmed.

**Limits:**
- 2008 US apparel data. A two-week outcome window.
- Only ~289 incremental conversions to learn heterogeneity from.
- Assumed economics: at $0.25 per contact the optimal depth drops to 35%, and at $0.50 mailing everyone loses $11,500.
- Seed instability: Qini −0.1 to 6.8, with only 44% overlap in selected customers across reruns.
- Not a production system: no serving, scheduling or monitoring.

## What I'd change at 10x scale

For a real CRM programme with 10x customers and repeated campaigns:
- **Continuous randomised holdouts** in every campaign, so uplift is always measurable.
- **A feature store** with customer history, and **scheduled re-scoring** before each send.
- **A policy service** that takes a budget and returns the list, with the frontier.
- **Longer outcome windows** (unsubscribes, long-term value) to find real sleeping dogs.
- **Many more positive outcomes** (more campaigns or larger files), because the data, not the algorithm, is the constraint.
- **Validation tests before rollout**, as the memo proposes (4 cells, ~82k customers).
