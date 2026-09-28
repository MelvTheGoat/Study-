# Stack ("Reckon"): 10 Points to Know by Heart

Repo: https://github.com/MelvTheGoat/Stack

1. **Reckon matches payments (card, dedicated account, transfer, cash) to invoices, closes what it can prove, and queues the rest.**
   *Why it matters:* a real, painful, daily fintech problem.

2. **The house rule: certain things first, scored things second, a model last. Every decision records its layer.**
   *Why it matters:* explainability and safety.

3. **Four deterministic steps: duplicate guard, exact reference, dedicated account + one invoice, exact amount in the window. None fires on a tie.**
   *Why it matters:* ~70% closed with pure facts.

4. **Model: 18 named features, a hand-written logistic regression, and Platt calibration.**
   *Why it matters:* readable weights and honest probabilities.

5. **Threshold 0.85 from costs: review ₦60 vs wrong auto-close ₦5,000 (83:1), chosen by 4-fold CV on 87 leftover payments.**
   *Why it matters:* a business decision expressed in naira.

6. **Results on the synthetic month (439 payments, ₦37.2M): 76.1% auto-closed, 0 wrong, recall 80.1%, and the first 40 reviews cover 91% of the money at risk.**
   *Why it matters:* know the numbers and that they're synthetic.

7. **Money is integer kobo everywhere. No floats in the money path.**
   *Why it matters:* exact books.

8. **Webhooks: HMAC SHA-512, store raw, 200 now, background task in its own session, the verify call is the authority, and status only moves forward.**
   *Why it matters:* real payment-integration experience.

9. **Review queue ordered by money at risk. An append-only audit log. Rejections become labels, with no automatic retraining.**
   *Why it matters:* human-in-the-loop design.

10. **494 tests pass with the optional LLM extra. 8 fail without it (the CI install). The LLM reader for prose reports is optional and not measured yet.**
    *Why it matters:* know the gaps before you're asked.
