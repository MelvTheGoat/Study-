# Credit-Risk-Decisioning: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

---

## 60-second pitch

> "I built a credit decisioning system rather than a default-prediction notebook. It produces calibrated probabilities of default, turns them into approve or decline at the lowest expected cost, explains each decline with adverse-action reason codes, audits fairness, and monitors drift. The headline is a cost, not an AUC. On an out-of-time test set, the cost-optimal cutoff of about 0.107, which comes from margin over margin-plus-LGD, beats an F1 cutoff by 730,000, and a 0.5 cutoff actually loses money.
>
> I built it on a synthetic book with ground truth for declined applicants, because that's the only way to check reject inference and fairness. That let me show reject inference hurts in one selection regime and helps in another, and that a group disparity was partly manufactured by biased income data. It's served by one decision engine behind a FastAPI service and a batch scorer, with an append-only audit log, integrity-checked model bundles, and CI that pushes to AWS ECR."

---

## Questions and honest answers

### 1. "Why LightGBM?"
It had the best net cost: 13.5% more profit than the scorecard on out-of-time data. But I decomposed the gap: monotonicity is free, coarse binning costs 1.73 AUC points, and non-linearity is only 0.58. A finer-binned scorecard recovers most of it, so if auditability matters more, the scorecard is a reasonable choice.

### 2. "How did you choose the cutoff?"
From the economics. Approving a good customer earns margin × EAD, and approving a bad one loses LGD × EAD. With calibrated PD, approve while `PD < margin/(margin+LGD) = 0.09/0.84 ≈ 0.107`. The empirical sweep gives 0.120, and the gap between them is the price of miscalibration under drift. F1 is the wrong objective, because it weighs both errors equally.

### 3. "How do you know the probabilities are right?"
Out-of-time Brier, ECE, calibration slope and intercept. The raw LightGBM was best (slope 1.035, ECE 0.0255). Isotonic looked perfect in-period but had a slope of 0.726 out-of-time. Every model under-predicts after the drift (0.145 vs 0.171), so the runbook says to recalibrate on recent matured outcomes.

### 4. "Why not use SMOTE for the imbalance?"
I tested it. Class weights, SMOTE and undersampling all destroyed calibration (ECE rose to 0.19–0.28), and SMOTE also lost ranking. The imbalance is one in eight, which is mild. And this output feeds PD × LGD × EAD, so calibration matters.

### 5. "What's reject inference, and does it work?"
We only see outcomes for approved people, so the model learns from a biased sample. I tested fuzzy augmentation, parcelling and Heckman. When the old policy only used visible features (MAR), the corrections tripled the bias. When a hidden signal drove approvals (MNAR), fuzzy augmentation removed almost all of it. You can't tell which case you're in from a real book, so I wouldn't apply it by default.

### 6. "Is your model fair?"
Not fully. Group B's adverse impact ratio is 0.755, below the four-fifths line, and the equal-opportunity gap is 13.6 points. Using the ground truth, I found 2.0 points of the PD gap are manufactured by under-recorded income for group B. I'd keep one cutoff (λ=0), escalate to model risk, and fix income capture, rather than use group-specific cutoffs, which are probably unlawful.

### 7. "How would you explain a decline?"
Points-shortfall reason codes from the scorecard: which characteristics fell furthest below their best points. They're reproducible forever from the points table. SHAP is available too, but it needs a versioned background set. Reasons based on `age` are flagged for legal review.

### 8. "How do you monitor it?"
PSI on the score and features, vintage default curves, and a combined-watch rule. In the test, PSI stayed below 0.25 while defaults rose 75%. Only the combined watch fired. Input drift is a prompt, not a safety net.

### 9. "What stops a bad deployment?"
The model and calibrator share a `training_run_id` and content hashes, and the service refuses to start on a mismatch. The API and batch use the same engine, with a byte-identical test. There's also a calibration regression gate in CI.

### 10. "How would you scale it?"
A separate stateless decision service with autoscaling, a feature store with point-in-time data, WORM audit storage, scheduled recalibration with champion/challenger, and monitoring dashboards with alerts.

### 11. "Why synthetic data?"
Because you can't validate reject inference or a fairness audit on real data: you never see declined applicants' outcomes. The synthetic book has ground truth for everyone. The trade-off is that no number transfers to a real portfolio. The UCI real-data path exists but hasn't been run on the real file.

### 12. "Do all your tests pass?"
With statsmodels below 0.15, yes: 227 pass. On a fresh install today, statsmodels 0.15 is picked up and 4 Heckman tests fail, because `predict(linear=True)` was removed. The fix is a one-liner or a version bound.

### 13. "What's wrong with your demo UI?"
Its default inputs are outside the training range. Income defaults to ₦5,000,000 but the synthetic book spans 9,000–900,000, and the loan default of ₦2,500,000 is way above the 200,000 max. It's also labelled in naira though the book has no currency. I'd align the form with the training distribution and warn on out-of-range inputs.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Synthetic data | "None of this is real." | "The methods and code are real. The numbers are properties of my generator. It was needed to test claims that can't be tested on real books." |
| Assumed economics | "Where does LGD 0.75 come from?" | "It's assumed. The sensitivity grid moves the cutoff from 0.053 to 0.211. A real system would model LGD and EAD." |
| `age` as a feature | "Isn't that discriminatory?" | "It's predictive and allowed in some contexts, but it's protected. Reasons based on it are flagged, and it needs a legal decision before launch." |
| statsmodels break | "Your tests fail." | "On statsmodels 0.15, 4 tests do. That's an unpinned dependency, and the fix is trivial." |
| Two processes in one container | "That's fragile." | "Yes. Fine for a demo. In production, the API and UI would be separate services with their own health checks." |
