# Uplift-Modelling-Decision: LinkedIn Post

*About 165 words. Copy from the line below.*

---

"The people we emailed bought more" doesn't tell you whom to email.

I ran an uplift-modelling study on the Hillstrom randomised e-mail trial (64,000 customers). The question: with a fixed budget, who buys *because* we contacted them?

What I found:
- On simulated data with a known true effect, a normal "likely to buy" model mailed 5× more customers the campaign actually harms than an uplift model did.
- A realistic non-randomised campaign list overstated the true effect by 7.5×.
- On the real trial, uplift targeting was worth about $1,390. Simply raising the budget was worth about $3,170. The budget was the real constraint.

The honest part: the targeting gain failed a placebo test on the main campaign. When I refit on shuffled treatment labels, the model sometimes scored higher than on real ones. The top 10% is real. The ranking below it isn't.

So the recommendation is a proper A/B test, not a rollout.

https://github.com/MelvTheGoat/Uplift-Modelling-Decision

#CausalInference #Marketing #DataScience #ABTesting
