# Loan Approval Decisions: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: the results come from a realistic **made-up** loan book, not real customers. That was done on purpose, and this file explains why.

## 1. The Problem

When someone applies for a loan, a lender must decide: approve or decline? Most people think the job is just "predict who won't pay back". But a real lender needs four things.

First, an honest chance that the person won't pay back. Second, a decision based on money: what a bad loan costs, and what a good one earns. Third, a clear reason for every decline, because the law often requires it. Fourth, proof that the decisions are fair to different groups of people.

This project does all four.

## 2. The Big Idea

The big idea is "the goal is money, not accuracy". So the main result is profit, not a score of how often the model was right.

A second big idea: test everything on a made-up loan book where the truth is known for *everyone*. In real life, a lender never learns what a declined person would have done, because they never got the loan. With made-up data, you know. So you can actually check things like fairness and bias corrections.

## 3. How It Works, Step by Step

**Training (done ahead of time):**

**Step 1: Make the loan book.** It creates 30,000 realistic loan applications over 36 months. Hidden rules decide who defaults (fails to pay back). From month 24, customers start to behave differently, on purpose. One group's income is recorded 15% too low, also on purpose, to test fairness.

**Step 2: Split by time.** Models learn from months 0 to 19, are fine-tuned on months 20 to 23, and are tested on months 24 to 35. That's like real life: you build on the past and use it on the future.

**Step 3: Learn only from approved loans.** Just like a real lender, the models only see repayments from people who were approved.

**Step 4: Train three models.** A classic scorecard (points for each answer on the form), a simple model called logistic regression, and LightGBM (a tool that builds many small yes/no flowcharts).

**Step 5: Make the chances honest.** The model's chances are checked against what really happened, and adjusted if needed. This is called calibration.

**Step 6: Set the cut-off line.** The line comes from money, explained below.

**Step 7: Run the extra studies.** These cover fairness, guessing what declined people would have done, and spotting changes over time.

**Step 8: Save the model package.** The model and its calibration are saved together, with a shared ID.

**Making a decision (live):**

**Step 9: An application arrives.** The decision service checks every field is in a valid range.

**Step 10: The engine decides.** It loads the model package, and refuses to start if the parts don't match. It works out the chance of default, the expected money loss, and approves or declines.

**Step 11: Give reasons.** Every decision comes with plain reasons. Reasons based on age are flagged for legal review, because age is protected by law.

**Step 12: Record it.** The decision is written to a permanent record before the reply is sent.

## 4. The Clever Parts

**The cut-off comes from money.** A good loan earns 9%. A bad loan loses 75% of the money. Approving is worth it while the expected gain beats the expected loss.

That balance point is 9 ÷ (9 + 75) ≈ 0.107. So approve if the chance of default is below about 11%. Using 0.5 instead, a common default, actually turned a profit into a loss.

**One decision engine for everything.** Both the live service and bulk scoring use the same code, so they can never disagree. A test checks their results are identical.

**Refusing mismatched parts.** A model paired with the wrong calibration still ranks people well, but its chances are wrong. That kind of mistake is invisible. So the service refuses to start if the parts don't match.

**Reasons you can reproduce.** Decline reasons come from where the applicant lost the most scorecard points. That's simple and gives the same answer every time, which matters for legal notices.

**Honest about bad news.** The fairness check found one group was approved at only 0.755 times the rate of another, below the common 0.8 warning line. The project shows this openly instead of hiding it.

## 5. The Important Words

- **Default**: failing to pay back a loan.
- **PD (probability of default)**: the chance someone will default, from 0 to 1.
- **LGD (loss given default)**: the share of the loan lost if someone defaults, here 75%.
- **Expected loss**: the average money you expect to lose on a loan.
- **Model**: a program that learns patterns from past examples.
- **Calibration**: making sure the model's chances are honest.
- **Cut-off**: the line where approve becomes decline.
- **Reason code**: a short, plain reason for a decline.
- **Fairness audit**: checking whether one group is treated worse than another.
- **Drift**: when customers or their behaviour change over time.

## 6. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **NumPy and pandas**: make and handle the data.
- **scikit-learn**: logistic regression, calibration and many measures.
- **LightGBM**: the best-performing model.
- **statsmodels**: one of the methods for guessing what declined people would have done.
- **SHAP**: explains how much each fact pushed a prediction up or down.
- **FastAPI**: the decision service.
- **Streamlit**: a demo form to try it out.
- **Docker, GitHub Actions and AWS ECR**: package, test and store the app.

## 7. How Good Is It?

These come from the last 12 months of the made-up book.

- **Profit**: LightGBM made about 385 per application. The scorecard made 13.5% less.
- **Ranking**: LightGBM's AUC was 0.84. AUC runs from 0.5 (guessing) to 1.0 (perfect).
- **The cut-off matters more than the model**: picking the wrong line cost far more than picking a slightly worse model.
- **Fairness**: full equality between the groups would cost 1.64% of profit.
- **Changes over time**: a common drift alarm (PSI) stayed quiet while defaults rose 75%. Only a combined check, which also watches real default rates, caught it.

## 8. What's Weak or Missing

- The data is made up, so the numbers won't transfer to a real lender.
- A path for real public data exists, but it was never run, because the download was blocked.
- The 75% loss and 9% profit are guesses. Across sensible values, the cut-off moves between 0.05 and 0.21.
- Age is both used by the model and protected by law, so it needs legal review.
- The permanent record is a local file, not special write-once storage.
- The demo form's default values are outside the range the model learned from.
- On a fresh install, 4 tests fail, because a newer version of statsmodels removed an option the code uses.

## 9. What This Project Shows You Can Do

- Think like a lender: decisions are about money, not accuracy.
- Make probabilities honest, and know why that matters.
- Measure fairness and bias openly.
- Spot changes over time with better alarms.
- Build one safe decision path, with a permanent record of every decision.

## 10. Ten Things to Remember

1. It decides whether to approve or decline loan applications.
2. The main result is profit, not accuracy.
3. It uses 30,000 made-up applications, where the truth is known for everyone.
4. It tests on later months, where customers change.
5. Three models are compared, and LightGBM made the most profit.
6. The cut-off is 9 ÷ (9 + 75) ≈ 11%, from money.
7. The cut-off matters more than the choice of model.
8. Every decline has plain reasons, and age-based reasons are flagged.
9. The fairness check found a gap, and the project shows it.
10. A common drift alarm missed a 75% rise in defaults.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
