# The cutoff is worth more than the model: building a credit decisioning system

Repo: https://github.com/MelvTheGoat/Credit-Risk-Decisioning

## Why I built it

Most credit-risk portfolio projects are a notebook that predicts default and reports an AUC. That's the easy fifth of the job. A lender approving unsecured consumer credit has four questions for every application, and only the first is a modelling problem:

1. **How likely is this person to default?** A *calibrated probability*, not a ranking, because expected loss is `PD × LGD × EAD` and that formula needs a real probability.
2. **Given what a default costs and what a lost customer costs, do we approve?** A *cost-optimal* cutoff, not an F1-optimal one.
3. **If we decline, what do we tell them?** Adverse-action reason codes that must still hold up months later.
4. **Are we being fair, and how would we know?** A fairness audit that reports the trade-off instead of hiding it.

So I built a system that answers all four, and made its headline number a **cost**, not an AUC.

## Why synthetic data first

The two hardest claims in credit modelling are that reject inference works and that a fairness audit finds real bias. Both are **unfalsifiable on real data**, because you never see what a declined applicant would have done.

So I wrote a generator: 30,000 applications across 36 monthly vintages, with a known default process (including a utilisation × delinquency interaction, a debt-to-income cliff and a thin-file step), deliberate drift from vintage 24, a known fairness violation (group B's *recorded* income understates their true income by 15%), a legacy approval policy, and **ground truth for everyone, including the declined**.

Everything is evaluated **out-of-time**: fit on vintages 0–19, calibrate on 20–23, test on 24–35, which include the drift.

## How it works

Three models sit behind one interface: a WOE points **scorecard** (the industry standard you can audit in a spreadsheet), an **L2 logistic regression**, and **LightGBM**. Probabilities are calibrated (raw, Platt or isotonic, picked by a rule that checks slope first and error second). Then a cost policy turns probability into a decision.

Serving goes through one `DecisionEngine`, called by both the FastAPI service and the batch scorer, so they can't drift apart. A test scores the same applicants both ways and requires byte-identical results. Every decision is written to an append-only audit log *before* the response is returned. A small Streamlit form sits on top for demos, and CI builds the Docker image, trains the model inside it, smoke-tests it, and pushes it to AWS ECR.

## What I found

### 1. The threshold is worth more than the model

Accounting is relative to writing no business: approving a good customer earns `margin × EAD`, approving a bad one loses `LGD × EAD`, and declining is zero. With LGD 0.75 and margin 9%, one missed bad costs as much as 8.3 good customers earn.

The best cutoff has a closed form that uses **no model at all**:

```
p*  =  margin / (margin + LGD)  =  0.09 / 0.84  =  0.1071
```

Same LightGBM model, four ways to pick the cutoff:

| Rule | Cutoff | Net cost |
|---|---|---|
| Cost-optimal (empirical) | 0.120 | −2,584,041 |
| Cost-optimal (closed form) | 0.107 | −2,506,563 |
| F1-optimal | 0.211 | −1,854,306 |
| Naive 0.5 | 0.500 | **+1,227,921** |

Negative is profit. Picking the cutoff by F1 costs **729,735**, more than twice the gap between the best and worst *models*. The 0.5 "default" doesn't just underperform. It **loses money**. F1 treats a false approval and a false decline as equally bad, which is a specific and wrong cost assumption applied silently.

And the cutoff is an assumption, not a measurement. Sweeping LGD from 0.45 to 0.90 and margin from 5% to 12% moves it from 0.053 to 0.211.

### 2. Scorecard vs booster: what exactly is lost

LightGBM beats the scorecard by 13.5% of profit. But "the scorecard loses 2.3 AUC points" isn't a finding until you know *what* it loses to:

| Variant | AUC |
|---|---|
| Scorecard, 8 monotonic bins (deployed) | 0.8148 |
| Scorecard, 8 bins, unconstrained | 0.8144 |
| Scorecard, 20 monotonic bins | 0.8227 |
| Logistic, continuous | 0.8321 |
| LightGBM | 0.8379 |

**Monotonicity is free** (0.0004 AUC). Coarse binning costs 1.73 points, half of which finer bins recover. All the non-linearity is worth 0.58 points. So most of the gap can be recovered *without* giving up the scorecard.

### 3. Accuracy means nothing here

LightGBM scores 0.857 accuracy at a 0.5 cutoff. Predicting "nobody defaults" scores 0.829. The operationally real metric is **precision at review capacity**: at 1% of files, 91% flagged are genuine bads.

### 4. Calibration: sometimes the fix is no fix

The raw LightGBM was already well calibrated out-of-time, so the selection rule kept it. Isotonic looked perfect on its own fitting data (ECE 0.0000) and made the model *more* overconfident out-of-time (slope 0.726). And every model under-predicted after the drift (0.145 vs 0.171 observed). No calibrator fitted before the drift can fix that, which is why the runbook says to refresh calibrators on recent outcomes.

### 5. Resampling bought nothing and cost everything

Class weighting, SMOTE and undersampling all wrecked calibration (ECE from 0.026 up to 0.28), and SMOTE also lost ranking. In a system that feeds `PD × LGD × EAD`, that's fatal, and invisible if you only look at AUC. The imbalance here (one bad in eight) doesn't need treating.

### 6. Reject inference depends on something you can't observe

The training data only has outcomes for **approved** applicants. I ran four correction methods under two selection regimes:
- **MAR** (the old policy used only visible features): corrections *tripled* the bias.
- **MNAR** (a hidden loan-officer signal drove approvals): fuzzy augmentation cut reject bias from −0.154 to −0.007.

And you can't tell which regime you're in from a real book, because the evidence is the missing counterfactual. Heckman removed 11% of the bias at best, even with a perfect exclusion restriction built for it.

### 7. Fairness: a real gap, and a manufactured one

No model sees `group`, yet group B is approved 49.2% of the time vs 65.1% for A (adverse impact ratio 0.755, below the four-fifths line). Because the ground truth exists, I could split the gap: the groups truly differ by 4.8 points of PD, and the model claims 6.8. **The extra 2.0 points are manufactured** by the income measurement bias. Dropping the protected attribute does nothing about it.

Full parity costs only 1.64% of profit. But I'd still deploy a single cutoff (λ=0), because group-specific cutoffs are probably unlawful and don't fix the probabilities underneath. The right fix is upstream: correct how income is recorded, then re-audit.

### 8. PSI missed it

Defaults in the last cohorts rose **75%**. Score PSI was 0.103 and the worst feature PSI 0.212, both under the usual 0.25 alarm. Only a "combined watch" rule (two indicators in the watch band together) fired. Input-drift metrics are a prompt, not a safety net.

## Explaining a decline

The system picks the decline closest to the cutoff as its worked example, because the marginal case is the one that has to be defensible. Application 27565 had a PD of 0.10715 against a cutoff of 0.10714, an expected value of −0.18 on a 17,600 exposure, and was declined with four points-shortfall reasons (low income relative to the loan, high debt-to-income, high utilisation, short employment). Reasons based on `age` are flagged `protected_basis` rather than printed silently.

## Safety in serving

A model and its calibrator carry the same `training_run_id` and content hashes. If they don't match, **the service refuses to start**, because a down service gets noticed in minutes, while one quietly serving wrong probabilities doesn't: AUC is unchanged by the mismatch.

## What I learned

- **Lead with cost.** It changes which model, cutoff and calibrator you pick.
- **Decompose gaps** before choosing a remedy.
- **Report results that go against the plan.** The scorecard didn't nearly match the booster, and reject inference didn't reliably help.
- **Fairness has a measurement layer.** Proxies and biased inputs matter more than the feature list.
- **Monitor outcomes, not just inputs.**

## Honest limits and what's next

Everything headline is on synthetic data. The UCI real-data path is implemented but was never run on the real file. LGD, EAD and margin are assumed. The audit log is a local file. There's no intersectional fairness analysis. And the Streamlit demo's default inputs sit outside the training range (e.g. income ₦5,000,000 against a synthetic range of 9,000–900,000), so I'd align the form with the data next. After that: real data, real LGD/EAD models, scheduled recalibration and WORM audit storage.
