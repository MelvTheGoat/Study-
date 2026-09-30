# Who to Target with Marketing: System Design for Beginners

This project answers a marketing question: with a fixed budget, which customers should get a campaign? The trick is that the goal isn't "who is likely to buy", but "who buys *because* we contacted them". It's a study, not a live service: a careful, repeatable analysis that ends in a short decision memo for a marketing director.

## Key Terms

- **Treatment**: the action being tested, here sending a marketing e-mail.
- **Control group**: customers who were randomly *not* sent the e-mail, for comparison.
- **Randomised trial**: an experiment where a coin flip decides who gets the treatment, so the groups are fair to compare.
- **Uplift**: the extra chance someone buys *because* they were contacted.
- **Model**: a program that learns patterns from past data to make a guess.
- **Sleeping dogs**: customers who are *less* likely to buy if you contact them.
- **Qini curve**: a chart showing how much extra buying you get as you contact more of the customers ranked highest.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Some customers buy anyway, and some never will. Money is only well spent on people who buy *because* of the e-mail, so that's who to find.

**Step 2: Figure out the data.** You can never see the same person both contacted and not contacted. So a randomised trial is needed, where a fair comparison is possible. This project uses a real e-mail trial with 64,000 customers.

**Step 3: Sketch the main parts.** First, test the methods on made-up data where the answer is known. Then apply them to the real trial, turn the results into a budget decision, and try hard to break the findings.

**Step 4: Walk through one decision.** Follow "should we e-mail the top 30% of customers?" from the models to the profit estimate.

**Step 5: Decide how to know it works.** Check whether targeting beats contacting people at random, with honest error ranges.

**Step 6: Plan for problems.** Small effects can be luck, models can find patterns in noise, and cost assumptions can be wrong. Plan a test for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Use a real e-mail trial of **64,000** customers.
- Assume each e-mail costs **$0.10**, and the profit margin on sales is **30%**.
- Plan for a budget that covers **30%** of customers.
- Show error ranges on **every** key number.

What would the e-mails cost? Step by step, for a list of 64,000:

1. 30% of 64,000 = **19,200** customers.
2. 19,200 × $0.10 = **$1,920** in e-mail costs.
3. So the targeting must bring in more than $1,920 of profit on top of what would happen anyway.

### The Big Picture (Step 3)

```
   [Made-up data with a known answer] --> [Check the methods work]
                                                   |
                                                   v
   [Real e-mail trial: 64,000 customers]
                   |
                   v
   [Uplift models rank customers]
                   |
                   v
   [Evaluate: does it beat random?]
                   |
                   v
   [Budget plan: profit at each size]
                   |
                   v
   [Stress tests] --> [Decision memo]
```

Step by step:

1. Made-up data is created where each person's true uplift is known, including a group of sleeping dogs.
2. Four uplift methods are tested on it, to check they can find the known answer.
3. The same methods are then used on the real trial, to rank customers by likely uplift.
4. The ranking is checked against random choice, with error ranges.
5. For each budget size, the real extra sales are *measured* from the trial's fair comparison, not guessed from the model.
6. Stress tests try to break the result, like shuffling the labels to see if "effects" still appear.
7. A two-page memo sums up the decision, the uncertainty, and what would change it.

### The Main Parts (Step 3)

**Made-Up Data First.** In real life you never learn one person's true uplift. So the methods are first tried on made-up data where the answer is known. It's like testing a metal detector on a beach where you buried the coins yourself.

**The Naive Mistake.** A quick comparison of people who got e-mails vs people who didn't can mislead badly. If the e-mail list wasn't chosen at random, the effect looked 7.5 times bigger than the truth. It's like judging a diet by only asking people who were already slim.

**Uplift Models.** Three "meta-learners" (S, T and X) are standard recipes for estimating uplift using ordinary prediction models. A fourth, the causal forest, comes from EconML (a Python tool for measuring cause and effect). Each gives every customer a likely-uplift score, like a weather forecast of "how much will an e-mail change this person's mind".

**Evaluation.** A normal accuracy score (AUC) is not used, because it rewards predicting *buyers*, not *persuadable* buyers. Instead, a Qini curve checks whether the top-ranked customers really gained more. A test then asks: is this better than choosing at random?

**Budget Plan.** For each budget size, profit is measured from the trial itself: extra sales among those contacted, minus e-mail costs. It's like checking the shop's till receipts, not the salesperson's forecast.

**Stress Tests.** A "placebo" test shuffles who got the e-mail and re-runs everything. If the method still finds big "effects" in shuffled data, the real result may be luck. Other tests change the costs and the random starting points.

Designing the follow-up experiment, including how big it must be, is its own topic (advanced - skip for now).

### How We Know It's Working (Step 5)

These come from the real trial's men's e-mail campaign.

- **Profit at 30% budget**: **$3,066** with targeting, against **$1,674** at random. But the error range is wide: $782 to $5,535.
- **Bigger budget vs better targeting**: raising the budget was worth about $3,170, and better targeting about $1,390.
- **Beats random?**: for men's sales, no method passed this test. For women's sales, two did.
- **Placebo test**: failed for the men's campaign, and passed for the women's.

### What Can Go Wrong (Step 6)

- **The gain is luck.** Only about 289 extra sales happened in the whole trial, which is very little to learn from. The failed placebo test warns about this.
- **Rankings change between runs.** With different random starting points, only 44% of the chosen customers stayed the same.
- **Wrong cost assumptions.** At $0.25 per e-mail, the best size drops to 35%. At $0.50, e-mailing everyone loses $11,500.
- **Old data.** The trial is from 2008, with a two-week window, so long-term harms like unsubscribes aren't seen.

## Quick Recap

- Target people who buy *because* of contact, not people who'd buy anyway.
- Test methods on made-up data with a known answer first.
- Measure profit from the fair comparison, not from the model's own guesses.
- Try hard to break the result, and report it when it fails.
- Here, a bigger budget mattered more than smarter targeting.
