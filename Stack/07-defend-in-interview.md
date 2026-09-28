# Stack ("Reckon"): Defending It in an Interview

Repo: https://github.com/MelvTheGoat/Stack

---

## 60-second pitch

> "Reckon matches incoming payments to invoices for a Nigerian business paid by card, dedicated account, bank transfer and cash, where most payments don't say what they're for. One rule runs it: certain things first, scored things second, a model last. Four exact rules close what can be proven, and none of them fires on a tie. What's left is scored by an 18-feature logistic regression, calibrated with Platt scaling, and it only closes above 0.85. That line comes from costs: a wasted review is ₦60 and a wrong auto-close about ₦5,000, so 83 to 1.
>
> Everything else goes to a review queue ranked by money at risk, and every decision goes into an append-only audit log. On a generated month of 439 payments, it closes 76% unattended with zero wrong, and the first 40 reviews cover 91% of the money at risk. The data is synthetic, so real-world numbers will be worse, and I say that up front."

---

## Questions and honest answers

### 1. "Why rules before a model?"
Rules are certain, cheap and explainable, and they do almost 70% of the work. The model is only needed for the ambiguous residual. Putting the model first would mean trusting a probability where a fact is available.

### 2. "Why doesn't a rule fire on a tie?"
Because a tie isn't certainty. If two invoices fit exactly, picking one by a rule is a guess dressed up as a fact. Those go down to scoring or to a person.

### 3. "How did you pick 0.85?"
From costs, not by eye. A review is 3 minutes at ₦1,200/hour = ₦60. A wrong close is 2 hours to unpick plus ₦2,600 of goodwill ≈ ₦5,000. At 83:1 you need to be wrong less than about 1.2% of the time. I chose the line by 4-fold cross-validation over the 87 leftover payments, and it came out at 0.85.

### 4. "87 payments? Is that enough?"
No. It's the honest use of a small corpus, but you'd want several hundred leftover cases to trust a threshold to two decimals. The costs are in one file, so re-running with real data moves the line automatically.

### 5. "How do you know the probabilities are right?"
Platt calibration, chosen over isotonic by Brier score on development data. On 390 held-out pairs the Brier score is 0.0219 and the ECE 0.024. The honest caveat is that most pairs are near zero. Only 8 held-out pairs are above 0.8, so calibration near the operating line has thin evidence.

### 6. "100% precision sounds too good."
It is, on real data. It's 0 wrong out of 334 on a generated corpus whose scenarios I wrote. I'd expect real errors, and the review queue's rejection reasons are where they'd show up first.

### 7. "Why not scikit-learn or gradient boosting?"
18 features and a few thousand rows. A hand-written logistic regression is readable end to end, installs with no compiler, and its weights line up with features a bookkeeper understands. That matters because the queue explains suggestions in words. If real data showed non-linear effects, I'd try a boosted model and compare calibrated cost.

### 8. "Why integer kobo?"
Floats can't represent 0.1 exactly, so books drift. Decimal allows a third of a kobo, which doesn't exist. Integers make impossible amounts impossible, and the conversions happen only at the edges, in one file, with tests.

### 9. "Why call Paystack's verify endpoint if the webhook is signed?"
A signature proves who sent it, not that it's still true. The webhook is a snapshot and the part attackers can aim at. The verify response wins on amount, fees and status. I also keep "not found" and "couldn't ask" as separate outcomes.

### 10. "What bug taught you something?"
The background worker couldn't find the webhook row it was handed. Starlette runs background tasks before tearing down a request-scoped dependency, so the shared session hadn't committed. It failed every time in tests. Now receive and process use separate sessions.

### 11. "How do you handle out-of-order webhooks?"
Status only moves forward by rank: pending, failed, success, reversed, refunded. A late `charge.success` can't un-refund a payment. The cost is that a reversal after a refund can't be expressed.

### 12. "Why rank the queue by money, not confidence?"
A person has about 40 reviews in an evening, and the goal is protecting the books. The first 40 by money cover 91% of the money at risk. The downside: small old items never rise. I'd add age escalation.

### 13. "Where's the LLM, and why is it off by default?"
Only on prose reports the patterns can't read. It's constrained to the same Pydantic schema, allowed to refuse, and never asked twice. It's off by default so the service never silently calls a paid API. Its accuracy isn't measured yet, because that needs a key.

### 14. "Do your tests all pass?"
With the optional LLM package installed, all 494 pass. With only the dev install, which is what CI uses, 8 LLM-reader tests fail with a missing module. That's a bug in how those tests are gated. They should be skipped when the extra isn't installed.

### 15. "How would you scale it?"
Postgres and a job queue, multi-tenant ledgers and thresholds, indexed candidate lookup, scheduled settlement pulls from Paystack, a second provider adapter, and human-approved retraining with drift monitoring.

---

## Weak spots and how to answer

| Weak spot | Poke | Answer |
|---|---|---|
| Synthetic data | "These numbers are made up." | "The payments are generated, and the method and code are real. Every assumption is in one file. Real data is the next step." |
| Thin calibration at the top | "Only 8 cases above 0.8?" | "Right. The high-confidence region needs more data before I'd trust it broadly." |
| Tests fail without the extra | "Is CI red?" | "It would be for 8 LLM tests. The fix is to skip them when the package is missing." |
| Model closes only 6% | "Why have a model at all?" | "It's the 6% that's safe to close at 83:1 costs. With real data and more history features, that share could grow." |
| Twin invoices auto-closed | "You said no ties." | "The rule layer refuses them. The scoring layer closes them, and the answer key accepts either invoice. In real life I'd check that's acceptable for the business." |
| Threshold vs theory | "At 83:1, calibrated theory says close above ~0.988. Why 0.85?" | "0.85 is what minimised the measured cost across cross-validated folds, where nothing between 0.85 and 0.988 was wrong. With so few high-score cases, I'd move it towards theory until real data proves 0.85 safe." |
| Not deployed | "Is it live?" | "Not yet. The deploy script refuses to ship an image that fails the smoke test." |
