# Credit-Risk-Decisioning: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

1. **A decisioning system, not a notebook: calibrated PD → cost-optimal decision → reason codes → fairness audit → monitoring → audit log.**
   *Why it matters:* it frames you as someone who builds for the business.

2. **Synthetic book: 30,000 applications, 36 vintages, drift from v24, injected income bias for group B, ground truth for rejects. Out-of-time split: fit v0–19, calibrate v20–23, test v24–35.**
   *Why it matters:* it explains why every claim can be checked.

3. **Cutoff p* = margin/(margin+LGD) = 0.09/0.84 ≈ 0.107. The F1 cutoff costs 729,735 more. A 0.5 cutoff loses money.**
   *Why it matters:* "the threshold is worth more than the model".

4. **LightGBM AUC 0.8379, net cost −2,506,563. Scorecard 0.8148 (−13.5% profit). Monotonicity is free, binning costs 1.73 points, non-linearity 0.58.**
   *Why it matters:* know the numbers and the decomposition.

5. **Calibration: raw LightGBM kept (slope 1.035, ECE 0.0255). Isotonic overfits (in-period ECE 0, OOT slope 0.726). All models under-predict after drift.**
   *Why it matters:* calibration is the backbone of expected loss.

6. **Resampling (weights, SMOTE, undersampling) destroyed calibration (ECE 0.19–0.28) with no ranking gain.**
   *Why it matters:* a common interview trap.

7. **Reject inference: hurts under MAR (bias tripled), helps under MNAR (−0.154 → −0.007), and Heckman managed only 11%.**
   *Why it matters:* nuanced, honest findings.

8. **Fairness: AIR 0.755, EOD 0.136, and 2.0 points of manufactured PD excess from income bias. Parity costs 1.64%. Recommend λ=0 plus fixing income capture.**
   *Why it matters:* shows judgement on fairness, not just metrics.

9. **Monitoring: PSI 0.103 (no alarm) while defaults rose 75%. Only the combined-watch rule fired.**
   *Why it matters:* monitor outcomes, not just inputs.

10. **Serving: one DecisionEngine for API and batch (byte-identical test), a bundle integrity check that refuses to start, append-only audit, Docker → AWS ECR. 227 tests pass on statsmodels <0.15, and 4 fail on 0.15.**
    *Why it matters:* production-minded, and aware of its gaps.
