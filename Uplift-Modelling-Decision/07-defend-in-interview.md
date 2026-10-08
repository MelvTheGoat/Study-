# Uplift-Modelling-Decision: Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

---

## 60-second pitch

> "I ran a budget-constrained uplift study on the Hillstrom randomised e-mail trial. The question: with a fixed budget, who converts *because* they were contacted, not just who's likely to convert. I validated four estimators, S-, T- and X-learners and a causal forest, on synthetic data where the true individual effect is known, and showed that a normal response model mails five times more harmed customers than an uplift model.
>
> On the real trial I evaluated with Qini curves rather than AUC, and measured policy profit from the randomised contrast inside each targeted slice. Targeting was worth about $1,390 at the current budget, but raising the budget was worth about $3,170. The key honest result: the men's campaign targeting failed a placebo test. The top decile is real, and the ranking below it isn't. So I recommended a four-cell validation experiment rather than a rollout, and sized it at about 41,000 customers per cell."

---

## Questions and honest answers

### 1. "What's uplift modelling?"
Predicting the *change* in outcome caused by an action for each person: P(buy | contacted) − P(buy | not contacted). It targets the persuadable, not the sure things, and it can flag people harmed by contact.

### 2. "Why not a normal response model?"
It ranks by who buys, so it spends budget on people who'd buy anyway. On synthetic truth, the response model's top 20% captured 440 incremental sales vs 657 for uplift, and it mailed 1,345 harmed customers vs 254.

### 3. "Why no AUC?"
AUC measures separating buyers from non-buyers. A perfect uplift model ranks sure-buyers low (zero uplift), so it can have poor AUC. I use Qini, AUUC and uplift by decile instead.

### 4. "How do you evaluate without knowing individual effects?"
Two ways. On synthetic data, compare to the known truth (PEHE, rank correlation). On real randomised data, use the Qini curve: in each top-k slice, compare treated vs control conversion rates. Randomisation makes that contrast causal.

### 5. "S vs T vs X-learner?"
S puts treatment in one model and tends to shrink effects (0.65× the true spread here). T fits separate models and inflates noise (1.69×). X imputes counterfactuals from the other arm and weights by propensity. It's better when arms are unbalanced (PEHE 0.088 vs 0.106 at 15/85).

### 6. "Did the model work on Hillstrom?"
Partly. The top decile's lift (1.36 pp, CI 0.71–2.02) is about twice the average. But deciles 2–10 show no trend. The men's placebo failed: shuffled-label runs averaged 2.8 ± 6.3 vs 9.4 real, and 2 of 10 were higher. In the committed leaderboard no estimator beats random on men's conversion (best p = 0.056). The women's campaign passed.

### 7. "So why recommend using the model at all?"
The top slice is genuinely more responsive, and the downside is small at these economics (every contact is profitable on average). But I framed it as a bet to test, and the memo recommends a four-cell experiment before operationalising.

### 8. "What's the biggest lever?"
The budget. At 30% of the file, profit is $3,066. At the profit-maximising 85%, it's $6,241. That's worth about $3,170, more than targeting's ~$1,390.

### 9. "How sensitive is this to assumptions?"
Very. At $0.25 per contact the optimal depth drops from 85% to 35%. At $0.50, mailing everyone loses $11,500. The first thing to confirm is the true cost per contact.

### 10. "What about sleeping dogs?"
The model flagged 18.6% of the men's file as harmed, but in the trial that group gained +0.44 pp (CI +0.02 to +0.87). So they're unprofitable (dropping them saves ~$680 in contact costs), not harmed. The two-week window may hide real long-term harm like unsubscribes.

### 11. "How would you validate this?"
Four cells: model-targeted vs business-as-usual, each with a 10% random holdback. Detecting 0.91% vs 0.68% needs ~40,900 per cell. The success criterion is set in advance: incremental profit per thousand contacts.

### 12. "What's CUPED, and would it help?"
Subtract the part of each outcome predictable from pre-period data to cut variance. It halved variance for visits, but only cut 1.7% for 1%-rate conversion. So it doesn't justify a smaller conversion test here.

### 13. "Why is the data the constraint?"
Only about 289 incremental conversions in the whole trial. Heterogeneity is hard to learn from that little signal, whatever the algorithm.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Placebo failure | "So your model is noise?" | "Below the top decile, on the men's campaign, yes. That's why I recommended a test, not a rollout." |
| Leaderboard fails beats-random | "No model beats random on men's conversion." | "Correct in the committed results. The top-decile lift is the part with a CI above zero." |
| Old data | "2008 is ancient." | "It's a method demonstration on a real randomised trial, not a forecast for today's inboxes." |
| Assumed costs | "Where's $0.10 from?" | "An input in config. The memo says to confirm it first, because it moves the answer more than the model does." |
| Single commit | "No history?" | "The study was pushed as one commit. The results files and tests show the work. The website came later, in normal commits." |
| Two implementations | "The site re-implements the maths in JavaScript. Won't it drift?" | "A parity test runs both on the same generated campaign and fails if any number disagrees." |
