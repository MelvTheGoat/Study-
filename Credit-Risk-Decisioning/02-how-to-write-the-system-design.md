# Credit-Risk-Decisioning: How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

---

## Step 1: Requirements (2 min)

One line:
> "Decide approve/decline on consumer loan applications at the lowest expected cost, with a legal reason for every decline, fair treatment, and an audit trail."

**Functional**
1. Score each application with a **calibrated** probability of default (PD).
2. Turn PD into approve/decline using the costs (LGD, margin, exposure).
3. Give adverse-action reason codes for declines.
4. Audit fairness across protected groups.
5. Monitor drift and performance by vintage.
6. Serve decisions by API and batch, and log each one.

**Non-functional**
- **Correct probabilities** (EL = PD × LGD × EAD).
- **Consistency:** API and batch give identical results.
- **Safety:** never serve a model with the wrong calibrator.
- **Traceability:** append-only audit.
- **Honest evaluation:** out-of-time, cost-based.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Applications (synthetic) | 30,000 over 36 vintages |
| Legacy approval rate | 70% |
| Default rate: all / approved / declined | 19.96% / 12.41% / 37.57% |
| Split | fit 12,095 / calibrate 2,389 / test 6,516 |
| Costs | LGD 0.75, margin 9%, EAD = loan amount |
| Cutoff | p* = 0.09/(0.09+0.75) ≈ 0.107 |

**Say:** "Low volume per decision. The hard parts are probability quality, cost and fairness."

---

## Step 3: High-level boxes (2 min)

```
[Applications] -> [Features] -> [Model: LGBM / logistic / scorecard] -> [Calibrator]
                                                                             |
                                                         [Policy: cost-optimal cutoff]
                                                                             |
                                                [Decision + reason codes + expected loss]
                                                        |                     |
                                                 [Audit log]          [API / Batch / UI]
Offline: [Out-of-time eval] [Reject inference study] [Fairness audit] [Monitoring: PSI + vintages]
Artifacts: [model + calibrator bundle, same training_run_id, hashed]
```

---

## Step 4: Deep dive (10 min)

### 4a. Data and validation
- Synthetic book with truth for declined applicants too. That's the only way to test reject inference and fairness.
- **Out-of-time** split by vintage, with drift in the test period.

### 4b. Models
- WOE scorecard (monotonic bins) as the auditable baseline, L2 logistic, LightGBM.
- Decompose the gap: monotonicity 0.0004 AUC, binning 1.73, non-linearity 0.58.

### 4c. Calibration
- Raw vs Platt vs isotonic. Pick by slope within 0.15 of 1, then lowest ECE.
- Never judge a calibrator on its own fitting data (isotonic: ECE 0.000 in-period, slope 0.726 out-of-time).

### 4d. Decision policy
- Approve good: +margin × EAD. Approve bad: −LGD × EAD. Decline: 0.
- Closed-form `p* = margin/(margin+LGD)`. The empirical sweep gives 0.120.
- F1-optimal costs 729,735 more. A 0.5 cutoff loses money.

### 4e. Explanations
- Points-shortfall reason codes (reproducible from the points table), and SHAP as an alternative.
- Flag protected-basis reasons (`age`).

### 4f. Fairness
- DPD, AIR (4/5 rule), EOD, and within-group ECE.
- Separate true risk gaps from model-made ones (group B: +2.0 points of excess).
- λ curve from one cutoff to parity. Recommend λ=0 and fixing income capture upstream.

### 4g. Serving and safety
- One `DecisionEngine` for API and batch.
- Bundle integrity check: refuse to start on a mismatch.
- Append-only, fsync'd audit before the response.

### 4h. Monitoring
- PSI on the score and features, and vintage default curves.
- A combined-watch trigger (two indicators in the watch band).

---

## Step 5: Bottlenecks and risks (2 min)

1. **Drift** breaks calibration. Every model under-predicts out-of-time (0.145 vs 0.171). Recalibrate on recent outcomes.
2. **Selection bias:** trained only on approved applicants.
3. **Assumed economics:** the cutoff ranges from 0.053 to 0.211 across LGD and margin assumptions.
4. **Slow truth:** vintage outcomes take months.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Cost-optimal cutoff | F1 / 0.5 | Money is the objective | Depends on assumed LGD/margin |
| LightGBM deployed | Scorecard | +13.5% profit | Less auditable (SHAP needs a background set) |
| No resampling | SMOTE / weights | Keeps calibration | — |
| λ=0 single cutoff | Group cutoffs | Legal, and fixes the root cause upstream | Disparity remains until fixed |
| Refuse on mismatch | Serve anyway | Loud failure beats silent miscalibration | Downtime |
| Synthetic ground truth | Real data only | Makes claims testable | Numbers don't transfer |
