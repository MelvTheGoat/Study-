# Who to Target with Marketing: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow the study: experiments, cause and effect, the uplift models, measuring them, turning results into a budget, testing the results, and planning the next experiment.

---

## 1. Experiments

### Treatment and control
**What it means:** the treatment group gets the action being tested, here an e-mail. The control group doesn't.

**Why it's needed here:** comparing the two shows what the e-mail actually changed.

### Randomised trial
**What it means:** an experiment where chance decides who gets the treatment.

**Why it's needed here:** with random assignment, the two groups are alike except for the e-mail. So any difference in buying is caused by the e-mail. The real data comes from such a trial.

### Hillstrom dataset
**What it means:** a well-known, public e-mail trial from 2008 with 64,000 customers, split into three groups: a men's e-mail, a women's e-mail, and no e-mail.

**Why it's needed here:** it's the real data the study uses. Its official link is often blocked, so the loader falls back to an identical copy online.

### Conversion
**What it means:** a customer buying something.

**Why it's needed here:** it's the main outcome measured. Men's e-mail conversion was 1.25%, against 0.57% with no e-mail.

### Selection bias
**What it means:** when the groups being compared differ in ways other than the treatment, because of how people were chosen.

**Why it's needed here:** the study shows that if the e-mail list isn't random, the effect can look 7.5 times bigger than it really is.

---

## 2. Cause and Effect

### Causal inference
**What it means:** working out what *causes* what, not just what goes together.

**Why it's needed here:** the question is whether the e-mail *causes* buying. That's a cause-and-effect question.

### Uplift (treatment effect)
**What it means:** how much more likely someone is to buy *because* they were contacted.

**Why it's needed here:** it's what the models estimate. Budget should go to the people with the highest uplift.

### ATE (average treatment effect)
**What it means:** the average uplift across everyone.

**Why it's needed here:** it's the overall effect of the campaign. Individual uplift varies around it.

### Heterogeneity
**What it means:** the effect being different for different people.

**Why it's needed here:** targeting only helps if some people respond more than others. If everyone responded the same, you'd just pick at random.

### Sleeping dogs
**What it means:** customers who become *less* likely to buy when contacted, as in "let sleeping dogs lie".

**Why it's needed here:** contacting them wastes money *and* loses sales. The models' "harmed" group in the men's campaign actually gained, so sleeping dogs weren't confirmed in the real data.

### Counterfactual
**What it means:** what would have happened under the other choice, which you can never see for the same person.

**Why it's needed here:** you can't see one person both e-mailed and not e-mailed. That's why made-up data with known answers is used to test methods first.

---

## 3. The Uplift Models

### Meta-learner
**What it means:** a recipe for estimating uplift using ordinary prediction models as building blocks.

**Why it's needed here:** three standard recipes are used: S, T and X. The building blocks can be swapped.

### S-learner
**What it means:** one model predicts buying, with "was e-mailed" as one of its inputs. Uplift is its prediction with the e-mail minus without.

**Why it's needed here:** it's simple and steady, but tends to shrink differences between people.

### T-learner
**What it means:** two separate models, one for e-mailed people and one for the rest. Uplift is the difference between them.

**Why it's needed here:** it's simple, but it tends to exaggerate differences, because each model's noise adds up.

### X-learner
**What it means:** a smarter recipe that uses each group's model to fill in the other group's missing outcome, then combines the results.

**Why it's needed here:** it works better when the groups are very different sizes.

### Causal forest
**What it means:** a method built from many decision trees, designed specifically to estimate uplift. This one comes from the EconML library.

**Why it's needed here:** it's a well-tested, specialised method to compare against the simpler recipes.

### Cross-fitting
**What it means:** scoring each customer with a model that didn't learn from that customer.

**Why it's needed here:** a model scoring people it learned from looks better than it really is. Cross-fitting keeps the evaluation honest.

### Propensity
**What it means:** the chance each person had of being treated.

**Why it's needed here:** in a randomised trial it's known, since the coin flip sets it. The code uses the known value rather than estimating it, which avoids adding noise.

---

## 4. Measuring the Models

### PEHE
**What it means:** the average error in estimating each person's true uplift. Lower is better.

**Why it's needed here:** it can only be measured on made-up data, where the truth is known. It's how the methods are checked first.

### Why not AUC
**What it means:** AUC is the usual score for how well a model ranks, like buyers above non-buyers.

**Why it's needed here:** it's deliberately *not* used. It rewards finding people who'll buy, not people the e-mail will *persuade*, which is the wrong goal.

### Qini curve and coefficient
**What it means:** the Qini curve shows extra purchases as you contact more of the top-ranked customers. The coefficient is the area between that curve and random choice.

**Why it's needed here:** it's the main measure of targeting quality. A bigger area means better targeting.

### AUUC
**What it means:** the area under the uplift curve, a close cousin of the Qini.

**Why it's needed here:** it gives a second view of the same question.

### Uplift by decile
**What it means:** splitting customers into 10 equal groups by predicted uplift, then measuring the real uplift in each.

**Why it's needed here:** it shows where the gain really is. Here, the top 10% had a clear lift, but the other 9 groups showed no pattern.

### Bootstrap confidence interval
**What it means:** re-running a calculation many times on resampled data to find a range of likely values.

**Why it's needed here:** it puts honest error ranges on every key number. The profit at 30% budget ranged from $782 to $5,535.

### Beats-random test
**What it means:** a statistical test asking whether the targeting is truly better than picking customers at random.

**Why it's needed here:** for men's sales, no method passed it. For women's sales, two did.

### p-value
**What it means:** roughly, how likely a result this good would be if there were really no effect. Below 0.05 is the usual bar.

**Why it's needed here:** the best method for men's sales got 0.056, just missing the bar.

---

## 5. The Budget Decision

### Policy
**What it means:** a rule for deciding who to contact.

**Why it's needed here:** here, the policy is "rank by predicted uplift, then contact the top X%".

### Budget frontier
**What it means:** a chart of profit at every possible budget size.

**Why it's needed here:** it shows the best size. With the assumed costs, profit peaked when contacting 85% of customers.

### Measured, not predicted, value
**What it means:** working out the gain from real results, not from the model's own guesses.

**Why it's needed here:** a model grading its own predictions is circular. Profit is measured from the randomised comparison within each chosen group.

### Unit economics
**What it means:** the cost and profit of a single unit, here one e-mail and one sale.

**Why it's needed here:** $0.10 per e-mail and a 30% margin are assumptions. Changing them changes the answer a lot.

---

## 6. Stress Tests

### Placebo test
**What it means:** re-running the analysis after randomly shuffling who got the e-mail. Any "effect" found then must be noise.

**Why it's needed here:** it's a hard check on whether results are real. The men's campaign failed it: 2 of 10 shuffled runs scored higher than the real one.

### Covariate balance
**What it means:** checking that the treatment and control groups look alike before the experiment, for example in age and past spending.

**Why it's needed here:** it confirms the trial was properly randomised. The groups were very well balanced.

### Seed stability
**What it means:** checking whether results stay the same when the random starting number changes.

**Why it's needed here:** here, they didn't. Only 44% of chosen customers overlapped between runs, showing the ranking is shaky.

### Sensitivity analysis
**What it means:** re-running with different assumptions to see how much the answer moves.

**Why it's needed here:** at $0.25 per e-mail, the best size drops to 35%. At $0.50, e-mailing everyone loses $11,500.

---

## 7. Planning the Next Experiment

### Statistical power and MDE
**What it means:** power is the chance an experiment detects a real effect. MDE (minimum detectable effect) is the smallest effect it can reliably detect.

**Why it's needed here:** it sizes the proposed follow-up test. Detecting 0.91% against 0.68% needs about 40,900 customers per group.

### CUPED
**What it means:** a trick that uses customers' past behaviour to reduce noise in an experiment, so smaller effects can be seen.

**Why it's needed here:** it's demonstrated here. It halved the noise for website visits, but barely helped for purchases.

### Decision memo
**What it means:** a short, plain-language document recommending a decision.

**Why it's needed here:** it's the final deliverable: two pages for a marketing director, covering the decision, the uncertainty, and what would change it.
