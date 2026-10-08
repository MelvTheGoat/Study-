# Who to Target with Marketing: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

One thing to know up front: this project is a **study**, not a running app. It's a careful, repeatable analysis that ends in a short decision memo for a marketing director.

## 1. The Problem

A company has a fixed marketing budget. It can only send an e-mail campaign to some of its customers. Who should get it?

Most people would say "the customers most likely to buy". But some of those would buy anyway, e-mail or not. Sending them an e-mail wastes money.

Some customers are even put *off* by being contacted. They're called "sleeping dogs", as in "let sleeping dogs lie".

## 2. The Big Idea

The right question is: who buys *because* we contacted them? The extra chance of buying caused by the e-mail is called **uplift**.

The project finds the customers with the highest uplift, works out how much profit that brings, and then tries hard to prove the result wrong. If the result doesn't survive the tests, the memo says so.

## 3. Why You Need a Fair Experiment

You can never see the same person both e-mailed and not e-mailed. So you can't directly measure one person's uplift.

The solution is a randomised trial. A coin flip decides who gets the e-mail. Because chance decided, the two groups are alike in every other way. So any difference in buying is caused by the e-mail.

This project uses a real, public randomised e-mail trial from 2008 with 64,000 customers. It's called the Hillstrom dataset. There was a men's e-mail, a women's e-mail, and a group that got no e-mail.

## 4. How It Works, Step by Step

**Step 1: Test the methods on made-up data first.** It creates made-up customers where each person's true uplift is known, including a group of sleeping dogs. It's like testing a metal detector on a beach where you buried the coins yourself.

**Step 2: Check the methods can find the answer.** Four uplift methods are tried on the made-up data, to see how well they recover the known truth.

**Step 3: Show the common mistake.** It shows that a quick comparison can mislead badly. If the e-mail list wasn't chosen at random, the effect looked 7.5 times bigger than the truth.

**Step 4: Use the real trial.** The same four methods give every real customer a likely-uplift score, and rank them.

**Step 5: Check the ranking beats random.** It asks: do the top-ranked customers really gain more than customers picked at random? Every number gets an error range.

**Step 6: Work out profit for each budget size.** For each size, like "top 30%", it measures the real extra sales from the trial itself, minus e-mail costs. It never trusts the model's own guesses for this.

**Step 7: Try to break the result.** These are the stress tests, explained below.

**Step 8: Plan the next experiment.** It works out how big a follow-up test would need to be.

**Step 9: Write the memo.** A two-page plain-language memo sums up the decision, the uncertainty, and what would change it.

## 5. The Clever Parts

**Made-up data first.** In real data, the true answer is never known. So the methods are checked first where it is.

**No normal accuracy score.** The usual score (AUC) rewards finding people who'll buy. That's the wrong goal. Instead, a "Qini curve" checks whether the top-ranked customers really gained more from the e-mail.

**Measured, not guessed.** Profit is measured from the fair comparison inside each chosen group. A model grading its own guesses would be like marking your own homework.

**The placebo test.** It randomly shuffles who got the e-mail, then re-runs everything. Any "effect" found in shuffled data must be pure luck. If the real result isn't clearly bigger than the shuffled ones, it might be luck too.

**Testing the assumptions.** It assumes each e-mail costs $0.10, and the profit margin is 30%. It then changes these to see how much the answer moves.

## 6. The Important Words

- **Treatment**: the action being tested, here an e-mail.
- **Control group**: customers who were randomly not sent the e-mail.
- **Randomised trial**: an experiment where chance decides who gets the treatment.
- **Uplift**: the extra chance someone buys *because* they were contacted.
- **Sleeping dogs**: customers who become less likely to buy when contacted.
- **Selection bias**: when compared groups differ for reasons other than the treatment.
- **Qini curve**: shows extra purchases as you contact more top-ranked customers.
- **Confidence interval**: an honest range of likely values for a number.
- **Placebo test**: re-running with shuffled labels to see if "effects" appear from luck.
- **Statistical power**: the chance an experiment detects a real effect.

## 7. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **NumPy, pandas and SciPy**: maths, tables and statistical tests.
- **scikit-learn and LightGBM**: ordinary prediction models used as building blocks.
- **EconML**: provides the "causal forest", a specialised uplift method.
- **pytest**: runs 150 automatic checks on made-up data.
- **ruff and mypy**: keep the code tidy and correct.
- **GitHub Actions**: checks every change automatically, and publishes the website.
- **Plain HTML, CSS and JavaScript**: the website, with no extra tools needed. It's hosted free on GitHub Pages.

## 8. How Good Is It?

These come from the real trial's men's e-mail campaign.

- The e-mail clearly worked overall: 1.25% of e-mailed men bought, against 0.57% with no e-mail.
- At a 30% budget, targeting made about **$3,066** profit, against **$1,674** at random. But the honest range was wide: $782 to $5,535.
- **A bigger budget mattered more than smarter targeting.** Raising the budget was worth about $3,170, and better targeting about $1,390.
- The top 10% of ranked customers had a clear extra lift. The other 90% showed no clear pattern.

But there are important warnings:

- For men's sales, **no method** clearly beat picking at random. For women's sales, two methods did.
- The placebo test **failed** for the men's campaign. 2 out of 10 shuffled runs scored higher than the real one.
- The sleeping dogs weren't confirmed. The group the models flagged as "harmed" actually bought more.

## 9. What's Weak or Missing

- Only about 289 extra sales happened in the whole trial. That's very little to learn from.
- The data is from 2008, and only covers two weeks, so long-term effects like unsubscribes aren't seen.
- The rankings change a lot between runs. Only 44% of chosen customers stayed the same.
- The cost assumptions drive the answer. At $0.25 per e-mail, the best size drops to 35%. At $0.50, e-mailing everyone loses $11,500.
- It's a study, not a live system. There is now a public website (added October 2026) that explains the study and lets you upload your own campaign as a CSV file. The analysis runs in your browser, so the file is never sent anywhere. But nothing runs on a schedule or sends e-mails.

## 10. What This Project Shows You Can Do

- Ask the right question: "who is persuadable?", not "who will buy?"
- Use fair experiments to measure cause and effect.
- Check methods on known answers before trusting them.
- Try hard to break your own results, and report failures honestly.
- Turn analysis into a clear business decision.

## 11. Ten Things to Remember

1. Target people who buy *because* of contact, not people who'd buy anyway.
2. That extra chance is called uplift.
3. A randomised trial makes a fair comparison possible.
4. The real data is a 64,000-customer e-mail trial from 2008.
5. Methods are tested on made-up data with known answers first.
6. A non-random list made the effect look 7.5 times too big.
7. Profit is measured from the trial, not from the model's guesses.
8. At 30% budget: about $3,066 with targeting, against $1,674 at random.
9. A bigger budget mattered more than smarter targeting.
10. The men's placebo test failed, so the targeting gain may be luck.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
