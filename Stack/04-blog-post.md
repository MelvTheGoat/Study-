# Reckon: matching payments to invoices without trusting a model more than it deserves

Repo: https://github.com/MelvTheGoat/Stack

## Why I built it

A lot of Nigerian businesses get paid four different ways at once:

| How | What you know about it |
|---|---|
| **Card** | Everything. The customer paid through your own checkout, so your invoice number came with it. |
| **Dedicated account** | Whose money it is (each customer has their own account number), but not *which* invoice. |
| **Bank transfer** | A line of text, like `NIP/GTB/OKONKWO ADA/PAYMENT`. That's it. |
| **Cash** | Whatever the person at the counter typed. |

Every evening, someone sits down and matches payments to invoices by hand. And it goes wrong the same ways every time: the name is spelled differently, or reversed, or shortened. Someone paid part of an invoice, or a bit too much. One transfer covers three invoices. The same payment was entered twice. Money came from someone who owes nothing. Two invoices are for the same amount. The bank credit is a day late and smaller than sales because of fees.

I wanted a system that does the boring, certain part, and hands a person exactly the cases that need judgement, in the order that protects the most money.

## The one rule

> **Certain things first. Scored things second. A model last.**

Every decision records which layer made it, so what happened on any given day is something you can read, not something you assume.

## How it works

### Getting payments in, safely

Payments arrive as Paystack webhooks. The endpoint checks the HMAC SHA-512 signature, stores the raw body, and answers 200 straight away. The real work happens in a background task.

Two details here took actual debugging:

**The verify call is the authority, not the webhook body.** A signed webhook proves who sent it, not that it's still true. So before a payment counts, the system asks Paystack `GET /transaction/verify/{reference}`, and Paystack's answer wins on amount, fees and status. "Paystack says no such reference" and "we couldn't reach Paystack" are different outcomes and must never collapse into one branch.

**Receive and process use two separate sessions.** I measured that Starlette runs background tasks *before* tearing down a request-scoped dependency. So with a shared session, the background worker went looking for a row that hadn't been committed yet and silently found nothing. It failed every time in tests, and would have failed intermittently in production, which is worse.

Webhooks also arrive out of order. A refund can land before the charge it refunds. So a payment's status only ever moves forward: pending → failed → success → reversed → refunded. An event that would move it backwards is ignored.

### Money is an integer

Every amount is a whole number of **kobo**. ₦1,250.50 is `125050`. Floats can't hold 0.1 exactly, and `Decimal` lets you write a third of a kobo, which doesn't exist. Integers make impossible amounts impossible.

### Layer 1: four certain rules

1. **Already seen.** Same money, same payer, same day, invoice already settled. That's a double submission, so a person decides.
2. **Exact reference.** Our invoice number is in the structured data or the narration, and it matches a real invoice exactly.
3. **Dedicated account.** Money landed in one customer's own account number, and they have exactly one open invoice it could be.
4. **Amount and time window.** Exactly one open invoice in the last few days is for precisely this amount, to the kobo.

The key constraint: **none of these fires when two invoices fit.** A tie isn't certainty.

### Layer 2: a small, calibrated model

Whatever's left is scored. For each plausible (payment, invoice) pair, the system builds 18 named features, each one something a bookkeeper would check:

```python
FEATURE_NAMES: tuple[str, ...] = (
    "name_similarity",
    "surname_exact",
    "landed_in_their_account",
    "amount_exact",
    "amount_closeness",
    "underpaid",
    "overpaid",
    ...
    "we_have_seen_this_payer_before",
    "round_number_payment",
)
```

Name similarity was the hardest feature. It has to treat "OKONKWO ADA" as "Ada Okonkwo", "Muhammad" as "Mohammed", "Seun" as "Oluwaseun" and "Okonko" as "Okonkwo", and strip narration noise like `NIP/GTB/.../PAYMENT`. But it must **not** merge two different customers, "Chinedu Okafor" and "Chinedum Okafor", who are in the test corpus on purpose. A similarity function that calls them the same is confidently wrong about whose money it is.

The model is a logistic regression written out by hand, with no scikit-learn. With 18 features and a few thousand rows, having the fit, calibration and scoring readable in one file was worth more than speed. Scores are then calibrated. Platt scaling and isotonic regression were both fitted, and Platt won. On 390 held-out comparisons, the Brier score is 0.0219 and the expected calibration error is 0.024.

### Where to draw the line

This is the part I like most. The threshold isn't a statistical choice. It's a business one, and it only has an answer once you say what each mistake costs:

| Mistake | Cost | Reasoning |
|---|---|---|
| Checking a match that was fine | **₦60** | 3 minutes at ₦1,200/hour |
| Closing a match that was wrong | **₦5,000** | 2 hours to notice, trace and reverse, plus ₦2,600 for chasing a customer who'd already paid |

That's about **83 to 1**. At that ratio, closing unattended only pays when you're wrong less than about 1.2% of the time. Cross-validation over the leftover cases puts the line at **0.85**. The costs live in one file, with the reasoning. Change them and re-run `make eval`, and the line moves on its own.

### Layer 3: people, in the right order

Everything else goes to a review queue **ordered by money at risk**, not by confidence. Each case arrives with a suggestion and the reasons in plain words. Rejections are stored as labels for a future training run, but nothing retrains itself overnight. A model that learns unattended from its own queue can drift somewhere strange without anyone noticing.

## The numbers

Everything comes out of `make eval`, which rebuilds the corpus, refits the model and re-scores from scratch. When I re-ran it, I got exactly the README's numbers.

**439 payments over one month, worth ₦37,181,750.67:**

| Layer | Resolved | Share | Right |
|---|---:|---:|---:|
| Exact reference | 164 | 37.4% | 100% |
| Dedicated account | 25 | 5.7% | 100% |
| Amount + window | 118 | 26.9% | 100% |
| Model | 27 | 6.2% | 100% |
| **Sent to a person** | **105** | **23.9%** | — |

- **76.1% closed without a person**, with zero wrong closes. Recall is 80.1%.
- The 105 sent to a person are the right ones: part payments, overpayments, splits, duplicates and money from nowhere. **Not one duplicate and not one stranger's payment was closed unattended.**
- A person gets through about 40 in an evening. **The first 40 cover 91% of the money at risk.**
- Against doing all 439 by hand: 16.7 hours and ₦20,040 saved a month, and that's the pessimistic end, since it assumes the human never makes a mistake.

## What it doesn't do

I wrote these down as things that are wrong with it today, not as "future work":

- **The data is simulated.** Every percentage moves if the scenario guesses are wrong. They're all in one place so you can disagree.
- **100% precision won't survive real data.** Watch the rejection reasons. That's where the first real errors will show up.
- **The threshold was chosen on 87 payments.** You'd want hundreds before trusting two decimal places.
- **Cash can't be verified.** If someone pockets ₦20,000 and records ₦15,000, this will reconcile cheerfully and be wrong.
- **Only Paystack is wired up**, so the "provider-neutral" seam hasn't been tested by a second provider.
- **The queue has no age escalation.** A ₦900 payment waiting three weeks stays at the bottom.

## Reading payment reports written by people

Some payments are reported in prose, like "Ada Okonkwo paid 45k for invoice 42 this morning by transfer". Pattern rules turn these into typed records or refuse. On 40 hand-labelled reports, patterns read 94.4% of the readable ones completely right, refused 100% of the unreadable ones, and invented nothing. But 40 examples written by the parser's own author means "it hasn't failed on these yet", not an accuracy rate. The 10 written *afterwards* found four real bugs straight away.

There's an optional last layer: a hosted LLM on the reports the patterns can't read, constrained to a JSON schema generated from the same Pydantic model, allowed to say "I can't read this", and never asked twice. It's off by default, and its numbers aren't measured yet.

## What I learned

- **Put certainty first.** Almost 70% of the month closes with plain rules that are easy to explain.
- **A threshold is a price.** Writing the costs in naira turned an argument into arithmetic.
- **Refuse ties.** Many wrong answers come from choosing between two equally good options.
- **Measure your framework's assumptions.** The session-ordering bug would have been a production mystery.
- **Rank human work by impact.** A small queue sorted well beats a big queue sorted badly.

## What's next

Real data, a second payment provider, pulling settlement batches from Paystack, age escalation in the queue, human-approved retraining from the labels, and scoring the LLM reading layer.
