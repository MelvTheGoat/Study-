# Loan Approval Decisions: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms are grouped by topic: lending basics, the data, the models, measuring them, making decisions, fairness, monitoring, and serving decisions.

---

## 1. Lending Basics

### Default
**What it means:** when a borrower fails to pay back their loan.

**Why it's needed here:** predicting default is the core job. Everything else builds on it.

### PD (probability of default)
**What it means:** the chance, between 0 and 1, that a borrower will default.

**Why it's needed here:** it's what the model outputs, and every decision is based on it.

### LGD (loss given default)
**What it means:** the share of the loan you lose if the borrower defaults.

**Why it's needed here:** it's assumed to be 75% here. Together with the profit margin, it sets the cutoff.

### EAD (exposure at default)
**What it means:** how much money is owed at the moment of default.

**Why it's needed here:** it's taken as the full loan amount. It's used to turn a chance into an expected money loss.

### Expected loss
**What it means:** the average amount you expect to lose on a loan: PD × LGD × EAD.

**Why it's needed here:** it's shown for every decision, so the risk is in money, not just a percentage.

### Margin
**What it means:** the profit made on a loan that's repaid.

**Why it's needed here:** it's assumed to be 9%. It's the other half of the cutoff sum.

### Vintage
**What it means:** a group of loans given out in the same month.

**Why it's needed here:** the book has 36 monthly vintages. Tracking them shows whether newer loans are going bad faster.

### Base rate
**What it means:** how common something is overall. Here, how many borrowers default.

**Why it's needed here:** about 17% default. At that rate, "accuracy" is misleading, because always saying "won't default" is right 83% of the time.

---

## 2. The Data

### Synthetic data
**What it means:** realistic, made-up data where the true answers are known.

**Why it's needed here:** real lenders never find out what a declined person would have done. With synthetic data, the truth is known for everyone, so fairness and bias corrections can actually be checked.

### Data drift
**What it means:** when the kind of customers or their behaviour changes over time.

**Why it's needed here:** it's built in on purpose from month 24. Testing on those months shows whether the model holds up when things change.

### Out-of-time split
**What it means:** training on older data and testing on newer data, instead of mixing them randomly.

**Why it's needed here:** that's how a real lender uses a model: built on the past, used on the future. Here the model learns from months 0 to 19, is adjusted on months 20 to 23, and is tested on months 24 to 35.

### Selection bias
**What it means:** when the data you learn from isn't a fair sample, because of how it was chosen.

**Why it's needed here:** a lender only sees repayments from people it approved. So the model learns from approved applicants only, just like in real life.

### Reject inference
**What it means:** methods that try to guess what declined applicants would have done, to correct selection bias.

**Why it's needed here:** the project tests four methods. They made things worse in one setup and better in another, and a real lender can't tell which setup they're in.

### MAR and MNAR
**What it means:** two kinds of missing data. MAR (missing at random) means the missing outcomes can be explained by data you have. MNAR (missing not at random) means they depend on something hidden.

**Why it's needed here:** the synthetic book includes a hidden loan-officer judgement, which creates the MNAR case. That's what makes reject inference so hard.

### Proxy
**What it means:** a feature that indirectly reveals something else, like a postcode hinting at ethnicity.

**Why it's needed here:** even without using a protected trait directly, a model can pick it up through proxies. That's why fairness has to be measured.

---

## 3. The Models

### Scorecard
**What it means:** a classic lending model that gives points for each answer on the form. The total points decide the result.

**Why it's needed here:** it's the industry standard, and easy to check in a spreadsheet. It's the baseline to beat.

### WOE (weight of evidence) binning
**What it means:** grouping a value (like income) into ranges, then scoring each range by how risky it is.

**Why it's needed here:** it's how the scorecard's points are made. The ranges are kept "monotonic", meaning risk only goes one way as the value rises.

### Logistic regression
**What it means:** a simple model that gives each feature a weight, adds them up, and turns the total into a probability.

**Why it's needed here:** it's a strong, simple middle option between the scorecard and LightGBM.

### LightGBM
**What it means:** a tool that builds many small yes/no flowcharts, each fixing the last one's mistakes.

**Why it's needed here:** it can catch interactions, like "high card use *and* missed payments". It made the most profit.

### Feature
**What it means:** one fact about an applicant, as a number, like income or debt-to-income ratio (DTI).

**Why it's needed here:** models learn from features.

### Class imbalance and resampling
**What it means:** class imbalance is when one outcome is much rarer than the other. Resampling changes the training data to balance them, for example SMOTE, which invents extra examples of the rare class.

**Why it's needed here:** the project tested resampling and found it ruined the probabilities without improving the ranking. So it isn't used.

---

## 4. Measuring the Models

### AUC (ROC-AUC) and Gini
**What it means:** AUC is how well a model ranks risky people above safe ones, from 0.5 (guessing) to 1.0 (perfect). Gini is the same idea on a different scale.

**Why it's needed here:** it measures ranking. LightGBM scored 0.84.

### KS statistic
**What it means:** the biggest gap between how the model scores good and bad borrowers.

**Why it's needed here:** it's a standard lending measure of how well the model separates the two groups.

### Calibration
**What it means:** whether predicted chances match reality.

**Why it's needed here:** expected loss needs honest chances. A model can rank perfectly and still give wrong chances.

### Platt scaling and isotonic regression
**What it means:** two ways to adjust a model's chances so they match reality. Platt uses a smooth curve, and isotonic uses a step shape.

**Why it's needed here:** the project tests both and picks the best. Isotonic looked perfect on its own data but did worse on later months. On this book, the raw LightGBM needed no adjustment.

### Brier score and ECE
**What it means:** two scores for how honest probabilities are, where lower is better. ECE (expected calibration error) is the average gap between predicted and real rates.

**Why it's needed here:** they're used to pick the calibration method.

### Calibration regression gate
**What it means:** an automatic test that fails if calibration gets worse than a saved baseline.

**Why it's needed here:** a code change could quietly break the probabilities. This gate catches it before release.

---

## 5. Making Decisions

### Cutoff (threshold)
**What it means:** the line where approve becomes decline.

**Why it's needed here:** it's set at 0.107 from costs, using margin ÷ (margin + LGD). The project shows the cutoff matters more than the choice of model.

### Cost-optimal
**What it means:** the choice that makes the most money (or loses the least).

**Why it's needed here:** a cutoff chosen for other reasons (like 0.5, or a common score called F1) cost far more money.

### Sensitivity analysis
**What it means:** re-doing a calculation with different assumptions, to see how much the answer moves.

**Why it's needed here:** the 75% and 9% are guesses. Across sensible values, the cutoff moves between 0.05 and 0.21.

### Reason codes (adverse action)
**What it means:** plain reasons given to a declined applicant. "Adverse action" is the legal term for a decline.

**Why it's needed here:** lenders must explain declines. Here, reasons come from where the applicant lost the most scorecard points, so they're always reproducible.

### SHAP
**What it means:** a method that shows how much each feature pushed a particular prediction up or down.

**Why it's needed here:** it explains the LightGBM model's predictions. The live reasons use scorecard points instead, which are easier to reproduce later.

---

## 6. Fairness

### Protected attribute
**What it means:** a personal trait the law protects from discrimination, like age or ethnicity.

**Why it's needed here:** the audit checks outcomes for different groups. Age is both a feature and protected, so reasons based on it are flagged.

### AIR (adverse impact ratio)
**What it means:** one group's approval rate divided by another's. Below 0.8 (the "four-fifths rule") is a common warning sign.

**Why it's needed here:** it was 0.755 between the two groups, below the warning line.

### Demographic parity and equal opportunity
**What it means:** demographic parity compares approval rates between groups. Equal opportunity compares approval rates among people who would actually repay.

**Why it's needed here:** they give different views of fairness, so both are measured.

### Model excess
**What it means:** how much more risk the model gives a group than their true risk.

**Why it's needed here:** one group's income was recorded 15% too low on purpose. The model gave them about 2 percentage points of extra predicted risk, which is unfair bias from the data.

### Fairness trade-off curve
**What it means:** a chart showing how much profit you give up for more equal outcomes.

**Why it's needed here:** full equality would cost 1.64% of profit. The project recommends not using different cutoffs per group, as that's likely unlawful.

---

## 7. Monitoring

### PSI (population stability index)
**What it means:** a number showing how much the mix of scores or features has shifted since training.

**Why it's needed here:** it's the usual drift alarm. Here it stayed quiet while defaults rose 75%, showing why it isn't enough alone.

### Combined watch
**What it means:** an alarm that looks at several signals together, including actual default rates by vintage.

**Why it's needed here:** it was the only check that caught the real problem.

---

## 8. Serving Decisions

### Model bundle (artifact)
**What it means:** a saved package of everything needed to make decisions: the model, its calibration and its settings.

**Why it's needed here:** the parts share one training ID and fingerprints. If they don't match, the service refuses to start, because a mismatch silently breaks the chances.

### Decision engine
**What it means:** the one piece of code that turns an application into a decision.

**Why it's needed here:** both the live API and batch scoring use it, so they can never disagree. A test checks their results are identical.

### API
**What it means:** a way for programs to ask for something, here "give me a decision for this application".

**Why it's needed here:** it's how applications come in. It checks every field is in a valid range first.

### Batch scoring
**What it means:** scoring many applications at once, rather than one at a time.

**Why it's needed here:** lenders often re-score a whole portfolio. It uses the same engine as the API.

### Audit log (append-only)
**What it means:** a permanent record where entries can be added but never changed.

**Why it's needed here:** every decision must be traceable later. It's written before the reply is sent. It's a local file for now, not special write-once storage.

### CI/CD
**What it means:** robots that automatically test the code (CI) and ship it (CD) when it changes.

**Why it's needed here:** every change is linted, type-checked and tested, including the calibration gate. The app is then packaged and uploaded to Amazon's container storage.
