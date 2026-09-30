# Reckon, Payment Matching: System Design for Beginners

This project helps a Nigerian business match each payment it receives to the right invoice. Payments come by card, transfer and cash, and most don't say which invoice they're for. Reckon closes the matches it can prove or is nearly sure about, and sends everything else to a person, biggest amounts first. It's a great example of mixing simple rules, a small model and human judgement.

## Key Terms

- **Invoice**: a bill sent to a customer, saying how much they owe.
- **Reconciliation**: matching payments to invoices so the books add up.
- **Webhook**: an automatic message one service sends to another when something happens, like "a payment just arrived".
- **Kobo**: the smallest unit of the naira. ₦1 = 100 kobo.
- **Model**: a program that learns patterns from past examples to make a guess.
- **Probability**: a chance between 0 and 1. 0.85 means 85%.
- **Threshold**: the cut-off line. Above it, act automatically. Below it, ask a person.
- **Audit log**: a permanent record of every decision, which can't be edited.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Closing a payment wrongly costs far more than checking it by hand. So the goal is "never close wrongly", then "save as much time as possible".

**Step 2: Figure out the data.** Payments arrive from Paystack (a payments company), plus informal reports like WhatsApp messages. Real business data can't be published, so the project generates a realistic month of payments, with an answer key.

**Step 3: Sketch the main parts.** Payments come in and are checked. Certain rules go first, then a scoring model, then a review queue for people.

**Step 4: Walk through one payment.** Follow one bank transfer from "alert received" to "matched" or "sent for review".

**Step 5: Decide how to know it works.** Measure how many payments close automatically, and whether any closed wrongly.

**Step 6: Plan for problems.** Alerts can arrive out of order, names are misspelled, and one transfer can pay several invoices. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Handle a month of about **440** payments, worth about **₦37 million**.
- Close a payment automatically only if it's certain, or the model is at least **85%** sure.
- Put everything else in a review queue, **biggest money first**.
- Record **every** decision permanently.

Why 85%? It comes from costs. Step by step:

1. A person costs about ₦1,200 an hour, which is ₦20 a minute.
2. A review takes about 3 minutes: 3 × ₦20 = **₦60**.
3. A wrong automatic close costs about **₦5,000** to find and fix.
4. ₦5,000 ÷ ₦60 ≈ **83**. A mistake costs about 83 reviews, so the model must be very sure before acting.

### The Big Picture (Step 3)

```
   [Payment alerts + written reports]
                 |
                 v
   [Receive, verify and read]
                 |
                 v
   [Payment record: amount in kobo, name, time]
                 |
                 v
   [Certain rules] --closed--> [Audit log]
                 |                  ^
                 v                  |
   [Scoring model] --closed---------+
                 |                  |
                 v                  |
   [Review queue] --decided---------+
                                    |
                            [Daily report]
```

Step by step:

1. Paystack sends a webhook saying a payment arrived. Reckon checks its signature, saves it, and replies "got it" straight away.
2. It then asks Paystack directly to confirm the payment, and Paystack's answer wins.
3. The payment becomes a simple record: amount in kobo, payer's name, time and channel.
4. The Certain Rules try first: an exact invoice reference, or exactly one invoice of that exact amount. If there's a tie, they step back.
5. If no rule fits, the Scoring Model checks every likely invoice and closes the match only if it's at least 85% sure, with a clear winner.
6. Everything else goes to the Review Queue.
7. Every close and every human decision is written to the Audit Log, and a Daily Report sums up the day.

### The Main Parts (Step 3)

**Receive and Verify.** A webhook could be faked, so its signature is checked, and Paystack is asked again before the payment counts. It's like phoning the bank to confirm a transfer instead of trusting a screenshot. A payment's status can only move forward, so a late-arriving old message can't undo a refund.

**Money in Kobo.** Every amount is stored as a whole number of kobo. Decimals in computers can drift by tiny amounts, which is unacceptable for money. It's like counting coins instead of estimating a pile.

**Certain Rules.** These close only what can be proved, like an exact invoice reference. If two invoices fit equally, the rules refuse to guess. It's like a teacher only marking answers that exactly match the answer sheet.

**Scoring Model.** For each possible invoice, it checks 18 simple facts a bookkeeper would check, like "does the name match?", "is the amount exact?" and "have we seen this payer before?". A logistic regression (a simple model that turns weighted facts into a probability) combines them into a chance. It's like an experienced bookkeeper's gut feeling, written down as numbers.

**Review Queue.** Unclear payments wait here, sorted by money at risk, each with a suggested match and plain reasons. A person approves or rejects. It's like a to-do list where the most expensive jobs are at the top.

**Audit Log.** Every decision records what came in, which part decided, the score, the evidence, when, and who. It's like a notebook written in pen, where nothing can be rubbed out.

Checking that "85% sure" really means 85% is called calibration (advanced - skip for now).

### How We Know It's Working (Step 5)

All these come from the generated month of payments. One command rebuilds every number, and they come out the same each time.

- **Closed without a person**: 76% of payments (rules 70%, model 6%).
- **Wrong closes**: **0** out of 334 automatic closes.
- **Review queue**: 105 payments, and the first 40 cover 91% of the money at risk.
- **Time saved**: about 17 hours a month.

The data is made up, so real-world results will be lower.

### What Can Go Wrong (Step 6)

- **Messages arrive out of order.** A refund alert can arrive before its payment. Status only moves forward, so the order doesn't matter.
- **Messy names.** Bank notes look like "NIP/GTB/OKONKWO ADA/PAYMENT". The name matcher removes noise words and handles spelling differences, but avoids merging two different customers with similar names.
- **Two invoices fit equally.** The certain rules refuse, and it goes to a person or the model.
- **Small old payments wait forever.** The queue sorts by money only, not age. This isn't fixed yet.

## Quick Recap

- Certain rules first, a careful model second, people last.
- The 85% line comes from real costs: a mistake costs about 83 reviews.
- Store money as whole kobo, and always confirm payments with Paystack.
- The review queue puts the biggest money first.
- Every decision is written in a permanent audit log.
