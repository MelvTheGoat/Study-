# Uplift-Modelling-Decision: 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Uplift-Modelling-Decision

1. **The question: with a fixed budget, who converts *because* of the campaign, not just who converts.**
   *Why it matters:* it's the core causal idea.

2. **Data: the Hillstrom randomised e-mail trial, 64,000 customers, 3 arms. 42,613 analysed. Men's e-mail 1.25% vs control 0.57%.**
   *Why it matters:* randomisation makes the contrast causal.

3. **Estimators were validated first on synthetic data with a known effect: S-learner shrinks (0.65×), T-learner inflates (1.69×), X-learner helps under imbalance.**
   *Why it matters:* trust is earned on known truth.

4. **The naive analysis misleads: selected lists overstate the effect 7.5×, and response models mail 5× more harmed customers than uplift models.**
   *Why it matters:* the business case for uplift.

5. **Evaluation uses Qini/AUUC/deciles with bootstrap CIs and a beats-random test. No AUC. Profit is measured from randomised contrasts, not predictions.**
   *Why it matters:* the correct way to evaluate uplift.

6. **At a 30% budget: $3,066 (CI $782–$5,535) vs $1,674 random. Optimum 85% → $6,241. Budget ≈ $3,170 vs targeting ≈ $1,390.**
   *Why it matters:* the budget is the bigger lever.

7. **The men's placebo test failed (9.4 vs 2.8 ± 6.3). The top decile is real (1.36 pp), and deciles 2–10 show no trend. Women's passed.**
   *Why it matters:* the honest headline caveat.

8. **The "harmed" group on the men's campaign actually gained +0.44 pp. It's unprofitable, not harmed. Sleeping dogs weren't confirmed.**
   *Why it matters:* don't over-read model predictions.

9. **Economics drive everything: at $0.25/contact the optimum falls to 35%, and at $0.50 mailing everyone loses $11,500.**
   *Why it matters:* confirm costs before acting.

10. **Next step: a 4-cell test with 10% holdbacks, ~40,900 per cell. CUPED helps visits (−50% variance), not conversion (−1.7%).**
    *Why it matters:* experiment design is part of the answer.
