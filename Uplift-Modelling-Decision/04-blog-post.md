# Who buys *because* of the campaign? An uplift study that reports its own failure

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

## Why I built it

Marketing teams love this sentence: "Customers we e-mailed converted at 1.25%, and those we didn't converted at 0.57%. The campaign works!" In the Hillstrom trial I used, that's true: a 119% lift, statistically overwhelming.

But it tells you almost nothing about **whom** to contact next time. With a fixed budget, the real question isn't "who is likely to buy?" It's "who buys *because* we contacted them?" Those are different people. Customers who were always going to buy look great to a normal model and add nothing. And some customers are *put off* by contact: the annoyed, the unsubscribers, the ones reminded to cancel. They're called "sleeping dogs".

I wanted to answer the budget question properly and write it up as a two-page memo a marketing director could act on (`MEMO.md`). Everything else in the repo exists to make that memo's numbers checkable.

## The data

**Hillstrom's MineThatData E-Mail Challenge (2008):** 64,000 customers, randomly split three ways: men's e-mail, women's e-mail, no e-mail. Randomisation is the key. It means the only systematic difference between the groups is the e-mail. The two arms I analysed hold 42,613 customers, and they're balanced on every covariate (the largest standardised difference is 0.014).

## Step 1: prove the tools work on known truth

On real data you never see a customer's individual effect. Each person is either mailed or not, never both. So a model that "looks good" on real data might just be reproducing your own mistakes.

I started with synthetic data where the true effect of contact is **known** for every customer, including a subgroup that's genuinely harmed. Then I tested four estimators:

- **S-learner:** one model, with "was contacted" as a feature. Its predicted effects came out at **0.65×** the true spread. It shrinks differences toward zero.
- **T-learner:** separate models for contacted and not contacted, then subtract. It came out at **1.69×**. It inflates noise.
- **X-learner:** fills in each customer's missing "other world" from the opposite group's model, then blends. It beat the T-learner when groups were unbalanced (error 0.088 vs 0.106 at a 15/85 split), as theory predicts.
- **Causal forest** (EconML): a forest designed to estimate effects directly.

All four recovered the known answer, which is what earned them the right to be used on the real data.

## Step 2: show why the obvious analysis misleads

Two demonstrations, both on simulated data where the truth is known:

1. **Selection bias.** Real campaign lists aren't random. They're built from engaged customers who'd buy anyway. Simulating that, a true effect of 2.2 percentage points showed up as a **16.9-point** gap: a **7.5× overstatement**.
2. **Response model vs uplift model.** Targeting the top 20%, a normal "likely to convert" model captured **440** truly incremental sales. The uplift model captured **657**. Worse, the response model mailed **1,345** customers the campaign harms, against **254** for uplift: five times as many.

## Step 3: evaluate without fooling myself

There's **no AUC** anywhere in this project. AUC measures how well a score separates buyers from non-buyers, and the best uplift model would score badly on it, because someone who always buys has zero uplift and should rank low.

Instead I used uplift metrics. The main one is the **Qini curve**: going down the ranked list, count extra conversions in the contacted group, adjusted for group size:

```
Q(k) = responders_treated(k) − responders_control(k) × n_treated(k) / n_control(k)
```

On top of that: the Qini coefficient, uplift by decile, bootstrap confidence intervals, and a one-sided test of whether the model beats random ordering.

The most important rule: **incremental effects are always measured from the randomised treated-vs-control comparison inside the selected group**, never read off the model's own predictions. Using predictions as values is circular, and it's the most common way uplift studies overstate themselves.

## Step 4: turn it into a decision

With $0.10 per contact and a 30% margin, here's profit from mailing the top N% by predicted uplift (men's e-mail):

| Share mailed | Profit | Profit per 1,000 contacts |
|---|---|---|
| 5% | $983 | $461 |
| 10% | $1,799 | $422 |
| **30% (current budget)** | **$3,066** | $240 |
| 70% | $6,115 | $205 |
| **85% (best)** | **$6,241** | $172 |
| 100% | $5,580 | $131 |

At the current 30% budget, model targeting makes **$3,066** (95% CI $782–$5,535) against **$1,674** for a random 30%. So targeting is worth about **$1,390**. But raising the budget to the profit-maximising depth is worth about **$3,170**. **The budget cap costs more than bad targeting does.**

## Step 5: try to break it (and it broke)

I ran four robustness checks. One failed, and it's the headline caveat of the memo.

**Placebo test:** I refitted the whole pipeline ten times on **randomly shuffled treatment labels**, where there's no effect to find. On the men's campaign, the real model scored **9.4**. The fake runs averaged **2.8** with a spread of **6.3**, and **two of the ten fake runs scored higher than the real one**. The men's ranking sits inside the range this method produces from pure noise. The women's campaign passed cleanly.

The rest of the evidence agrees:
- The **top decile is real**: a 1.36-point lift (CI 0.71–2.02) vs a 0.68 average. **Deciles 2–10 show no trend at all.**
- Across random seeds the Qini ranged from −0.1 to 6.8, and reruns picked only **44%** of the same customers.
- The model flagged 18.6% of the men's file as "harmed", but in the trial that group actually **gained** +0.44 points. They're unprofitable to mail, not harmed.

So the memo's recommendation is: send the men's creative, spend the whole budget, use the model for the top slice, and **treat the targeting gain as a bet to test, not a finding to roll out**.

## Step 6: design the test

A four-cell experiment: model-targeted vs business-as-usual, each with a random 10% holdback so incremental effects are *measured*. To detect 0.91% vs 0.68% conversion at 95% confidence and 80% power: **40,900 customers per cell, about five weeks**.

I also tested CUPED, a variance-reduction trick using pre-test behaviour. On a continuous metric (site visits) it halved variance. On 1%-rate conversion it cut **1.7%**, which is nothing. The memo warns not to quote the first number to justify a smaller conversion test.

## What I learned

- **"It worked on average" and "whom to target" are different questions.**
- **Validate estimators where the truth is known** before trusting them where it isn't.
- **Never grade a model with its own predictions.**
- **The business constraint can matter more than the model.** Here, the budget did.
- **A negative result, reported clearly, is worth more than a shaky positive one.**

## Limits

It's 2008 US apparel data with a two-week window, so long-term harm (fatigue, unsubscribes) is invisible. There are only about 289 incremental conversions to learn from. And the economics are assumptions: at $0.25 per contact the best depth drops from 85% to 35%, and at $0.50 mailing everyone loses $11,500.
