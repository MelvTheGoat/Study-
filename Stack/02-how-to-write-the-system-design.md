# Stack ("Reckon"): How to Write the System Design Yourself

Repo: https://github.com/MelvTheGoat/Stack

---

## Step 1: Requirements (2 min)

One line:
> "Match incoming payments to invoices automatically where we're certain, and send the rest to a person, ranked by money at stake."

**Functional**
1. Receive payments from Paystack (webhooks), plus cash and prose reports.
2. Match each payment to its invoice(s).
3. Close only what's certain or nearly certain. Queue the rest.
4. Let a person approve or reject, and learn labels from rejections.
5. Produce a daily settlement report (gross, fees, timing).
6. Keep an audit trail of every decision.

**Non-functional**
- **Never move money.** Test keys only, read and recommend.
- **Exact money:** integer kobo.
- **Correctness over coverage:** a wrong auto-clear costs about 83× a review.
- **Idempotent, order-safe webhooks.**
- **Explainable:** which layer decided, and why.

---

## Step 2: Numbers (1 min)

| Thing | Number |
|---|---|
| Payments / month (corpus) | 439, ₦37.2M |
| Channels | card, dedicated account, transfer, cash |
| Features per candidate | 18 |
| Review cost | ₦60 (3 min) |
| Wrong auto-clear cost | ₦5,000 |
| Threshold | 0.85 |
| Evening capacity | ~40 reviews |

**Say:** "Hundreds of payments a day at most. The hard part is correctness, not throughput."

---

## Step 3: High-level boxes (2 min)

```
[Paystack webhook] -> [Verify signature, store raw, 200] -> [Background: verify call, forward-only status]
[Prose report] -> [Patterns] -> [optional LLM, schema-bound]            |
                                                                         v
                                                    [Plain payment record (kobo)]
                                                                         |
            [Duplicate guard] -> [Exact ref] -> [Dedicated acct] -> [Exact amount window]
                                                                         |
                                             [18-feature logistic + Platt, p >= 0.85]
                                                    |                          |
                                               [closed]                 [review queue by ₦ at risk]
                                                    \__________ [append-only audit] __________/
                                                                         |
                                                              [daily settlement report]
```

---

## Step 4: Deep dive (10 min)

### 4a. Webhook ingestion
- HMAC SHA-512 signature, raw body stored, 200 immediately (idempotent on the event key).
- Background task with **its own session** (a shared request-scoped session hadn't committed yet).
- **Verify call is the authority**: "not found" → failed, "couldn't ask" → retry later.
- Status rank: pending < failed < success < reversed < refunded. Only move forward.

### 4b. Money
- `Money(kobo: int)`. Parsing raises on sub-kobo values. Conversions happen only at the edges.

### 4c. Deterministic layer
1. Duplicate guard (same money, payer, day, invoice already settled).
2. Exact reference (regex, then **checked against the live ledger**).
3. Dedicated account plus exactly one invoice of that amount.
4. Exactly one open invoice of that exact amount in the window.
- **Never fires on a tie.**

### 4d. Probabilistic layer
- Candidates: plausible open invoices.
- 18 named features (name similarity, surname exact, amount closeness, under/over, recency, channel, only open invoice, payer seen before...).
- Hand-written logistic regression → Platt calibration → close if ≥ 0.85.

### 4e. The threshold as a cost decision
- Close if `p_wrong × ₦5,000 < ₦60`, i.e. p_wrong < ~1.2%.
- Chosen by cross-validation over the residual.

### 4f. Review
- Order by money at risk. Suggestions with reasons in words.
- Rejections are stored as labels (`/api/labels`). No automatic retraining.

### 4g. Prose intake
- Patterns first. If unreadable, an optional LLM bound to the same Pydantic schema, allowed to refuse, asked once.

---

## Step 5: Bottlenecks (2 min)

1. **Out-of-order or duplicate webhooks:** handled by forward-only status and idempotency.
2. **Paystack unreachable:** the payment stays unverified and retryable.
3. **Review capacity:** ranked by money. The first 40 cover 91% of the money at risk.
4. **Scale:** single process. Postgres + a job queue for more.

---

## Step 6: Trade-offs (2 min)

| Chose | Over | Because | Cost |
|---|---|---|---|
| Rules first | Model first | Certainty is free and explainable | Rules need maintenance |
| High threshold (0.85) | More automation | 83:1 cost ratio | Model closes only 6% |
| Hand-written logistic | scikit-learn / GBM | 18 features, readable, no compiler | Less capacity |
| Generated corpus | Real data | Publishable, with an answer key | Numbers depend on assumptions |
| Verify call | Trust webhook body | Webhooks can be stale or spoofed | Extra call and failure mode |
| Integer kobo | float / Decimal | Impossible amounts can't exist | Conversions at boundaries |
| Queue by money | Queue by confidence | Protects the books | Small old payments wait |
