# Loan Approval Decisions: System Design for Beginners

This project decides whether to approve or decline loan applications. It doesn't just guess "will this person repay?". It gives an honest probability, makes a decision based on money, explains every decline, and checks the decisions are fair. It's trained on a realistic made-up loan book, where the true outcome is known for everyone, even people who were declined.

## Key Terms

- **Default**: when a borrower fails to pay back a loan.
- **PD (probability of default)**: the chance, from 0 to 1, that a borrower will default.
- **Model**: a program that learns patterns from past loans to make a guess about new ones.
- **Calibration**: whether the chances are honest. If the model says 10% for many people, about 10% of them should default.
- **Cutoff**: the line where "approve" becomes "decline".
- **Reason code**: a short, plain reason for a decline, like "high existing debt".
- **Fairness audit**: checking whether one group of people is treated worse than another.
- **Synthetic data**: realistic, made-up data where the true answers are known.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** The real goal is money: approve people who make a profit, and decline people who'd cost more than they bring in. So the headline result is profit, not accuracy.

**Step 2: Figure out the data.** Real lenders never learn what a declined person would have done. So the project builds a synthetic loan book with the truth for everyone. A path for a real public dataset exists, but it wasn't run.

**Step 3: Sketch the main parts.** A training programme builds the model. One decision engine serves every application, and every decision is logged.

**Step 4: Walk through one application.** Follow one application from the form to "approved" or "declined, because…".

**Step 5: Decide how to know it works.** Measure profit, how honest the chances are, and fairness between groups.

**Step 6: Plan for problems.** Customers change over time, some data is biased, and the model files can get mixed up. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Learn from **30,000** applications over **36** months.
- Test on the **last 12 months**, where the customers change, to see if it holds up.
- Assume a bad loan loses **75%** of the money, and a good loan earns **9%**.
- Explain **every** decline, and log **every** decision.

Where does the cutoff come from? Step by step:

1. For a loan of 100, a good borrower earns 9, and a bad one loses 75.
2. Approving is worth it while the expected gain beats the expected loss.
3. That balance point is: 9 ÷ (9 + 75) = 9 ÷ 84 ≈ **0.107**.
4. So approve if the chance of default is below about **11%**, and decline if it's above.

### The Big Picture (Step 3)

```
   [Training Programme] ---> [Model Bundle (versioned)]
                                     |
                                     v
   [Application] --> [Decision API] --> [Decision Engine]
                                              |
                                              v
                          [Chance of default + cost rule]
                                              |
                                              v
                          [Decision + reasons] --> [Audit Log]
```

Step by step:

1. Ahead of time, the Training Programme builds and tests models, then saves the chosen one as a Model Bundle.
2. An application arrives at the Decision API, which checks every field is in a valid range.
3. The Decision Engine loads the bundle, and refuses to start if its parts don't match.
4. The model gives a chance of default, which is compared with the 11% cutoff.
5. The engine works out the reasons for the decision.
6. Before replying, the decision is written to the Audit Log, a permanent record.
7. The same engine also scores whole batches of loans, so both routes always agree.

### The Main Parts (Step 3)

**Synthetic Loan Book.** It creates 30,000 realistic applications with hidden rules for who defaults. Customers change from month 24 onwards, and one group's income is recorded 15% too low on purpose. It's like a flight simulator: you can test risky situations safely because you know what really happened.

**Three Models.** A classic scorecard (points for each answer, easy to audit), a simple model called logistic regression, and LightGBM (a tool that builds many small yes/no flowcharts). Testing all three is like trying three routes to work and timing each one.

**Calibration.** The model's chances are checked against what really happened, and adjusted if needed. The lending maths depends on honest chances, so this matters more than ranking. It's like checking your kitchen scale against a known weight.

**Cost Rule.** Approve if the chance of default is below about 11%. Picking this line well matters more than picking the model. It's like a shopkeeper deciding the lowest price they'll accept, based on what it cost them.

**Reason Codes.** Every decline comes with plain reasons, based on where the applicant lost the most points. Reasons based on a protected trait, like age, are flagged for legal review instead of being printed. It's like a teacher explaining exactly where marks were lost.

**Audit Log.** Every decision is saved before the reply, with the inputs, score, reasons and model version. It's like a receipt book where pages can't be torn out.

Correcting for "we never see what declined people would have done" is called reject inference (advanced - skip for now).

### How We Know It's Working (Step 5)

These come from the last 12 months of the synthetic book.

- **Profit**: LightGBM made about 385 per application. The scorecard made 13.5% less.
- **The cutoff matters most**: using 0.5 instead of 0.107 turned a profit into a loss.
- **AUC**: how well the model ranks risky above safe, from 0.5 (guessing) to 1.0 (perfect). LightGBM scored **0.84**.
- **Fairness**: one group was approved at 0.755 times the rate of the other, below the common 0.8 guideline. The project shows this, rather than hiding it.

### What Can Go Wrong (Step 6)

- **Customers change.** A common drift alarm stayed quiet while defaults rose 75%. Only a combined check, which also watches actual default rates, caught it.
- **Mismatched model files.** A model paired with the wrong calibration still ranks well, but its chances are wrong. The engine refuses to start on a mismatch.
- **Biased data.** One group's income was understated, which pushed their predicted risk up. The fairness audit measures this gap.
- **Assumed costs.** The 75% and 9% are guesses, and the cutoff moves from 0.05 to 0.21 across sensible values.

## Quick Recap

- The goal is profit, so the headline result is money, not accuracy.
- The cutoff comes from costs: 9 ÷ (9 + 75) ≈ 11%.
- Honest (calibrated) chances matter more than a slightly better ranking.
- One decision engine serves every route, and every decision is logged.
- Fairness and drift are measured openly, including the bad news.
