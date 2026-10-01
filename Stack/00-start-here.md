# Reckon, Payment Matching: The Whole Project in Simple English

This file explains the whole project in simple English, from start to finish. Read it first. After this, the other files in this folder will be much easier to follow.

## 1. The Problem

Imagine a small Nigerian business. Customers pay by card, by bank transfer, into a special account, or in cash. Most payments don't say which invoice (bill) they're for.

Every evening, someone sits down and matches payments to invoices by hand. It goes wrong in the same ways every time. Names are misspelled, people pay part of a bill or too much, one transfer covers three invoices, payments arrive twice, and strangers send money.

## 2. The Big Idea

Reckon matches each payment to the right invoice automatically, but only when it's sure. It closes the matches it can prove, and the ones it's nearly certain about.

Everything else goes to a person, in a review list sorted by money at risk, biggest first. Every decision is written in a permanent record.

The order is: certain rules first, a careful scoring model second, and people last.

## 3. How It Works, Step by Step

**Step 1: A payment alert arrives.** Paystack, a Nigerian payments company, sends an automatic message called a webhook when a payment happens.

**Step 2: Check the alert is real.** Anyone could send a fake "payment successful" message. So Reckon checks a secret stamp (a signature) that proves it came from Paystack. It saves the message and replies "got it" straight away.

**Step 3: Double-check with Paystack.** After replying, Reckon asks Paystack directly to confirm the payment. Paystack's direct answer always wins over the alert.

**Step 4: Make a simple payment record.** The payment becomes a tidy record: amount, payer's name, time and how it was paid. Money is stored as whole kobo (₦1 = 100 kobo), never as decimals.

**Step 5: Try the certain rules.** The rules look for an exact invoice number, money landing in the customer's own special account with exactly one invoice of that amount, or exactly one open invoice for that exact amount. If any of these fit, the match is closed. If two invoices fit equally, the rules refuse to guess.

**Step 6: Try the scoring model.** If no rule fits, a model checks every likely invoice. It closes the match only if it's at least 85% sure and there's a clear winner.

**Step 7: Send the rest to a person.** Unclear payments go to the review list, with a suggested match and plain reasons. The biggest amounts go first.

**Step 8: Record everything.** Every close and every human decision is written in a permanent record that can't be edited. A daily report sums up the day.

## 4. The Clever Parts

**Why 85%? It comes from real costs.** A person costs about ₦1,200 an hour, so a 3-minute check costs about ₦60. Closing a match wrongly costs about ₦5,000 to find and fix.

So one mistake costs about as much as 83 checks. That's why the model must be very sure before acting.

**Checking like a bookkeeper.** The model looks at 18 simple facts for each possible match. Examples are "does the surname match exactly?", "is the amount exact?" and "have we seen this payer before?". Each one is something a bookkeeper would check, so the reasons can be explained in words.

**Messy names handled carefully.** Bank notes look like "NIP/GTB/OKONKWO ADA/PAYMENT". The name matcher removes noise words, handles spelling differences and short forms, and doesn't care about word order. But it's careful never to merge two different customers with similar names.

**Messages out of order.** A refund alert can arrive before the payment it refunds. A payment's status can only move forward, so an old message can't undo a newer state.

**Honest probabilities.** The model's chances are adjusted so that "85% sure" really means right about 85% of the time. This is called calibration.

**Written reports too.** Some businesses get payment reports as WhatsApp messages. Patterns pull out amounts, names, dates and invoice numbers. An optional AI reader handles the ones the patterns can't read. It's off by default, and only asked once.

**Made-up data with an answer key.** Real business payments can't be published. So the project generates a realistic month of payments, with the correct answer for every one. One command rebuilds every number in the results.

## 5. The Important Words

- **Invoice**: a bill sent to a customer.
- **Reconciliation**: matching payments to invoices so the books add up.
- **Webhook**: an automatic message one service sends another when something happens.
- **Kobo**: the smallest unit of the naira. ₦1 = 100 kobo.
- **Idempotent**: doing the same thing twice has the same effect as once, so a resent alert is never counted twice.
- **Model**: a program that learns patterns from past examples to make a guess.
- **Threshold**: the cut-off line, here 85%, for acting automatically.
- **Calibration**: making sure the model's chances are honest.
- **Audit log**: a permanent record of every decision that can't be edited.
- **Synthetic data**: realistic, made-up data where the right answers are known.

## 6. The Tools, in One Line Each

- **Python**: the language everything is written in.
- **FastAPI**: receives Paystack's alerts and serves the review pages.
- **SQLAlchemy**: stores everything in a database (SQLite, or PostgreSQL for bigger setups).
- **Pydantic**: defines what a payment record looks like.
- **httpx**: double-checks payments with Paystack.
- **Jinja2**: builds the review and report web pages.
- **Python's built-in tools**: the name matcher and the hand-written scoring model.
- **pytest, mypy and ruff**: keep the code correct and tidy.
- **Docker and Google Cloud Run**: package the app and run it online.

## 7. How Good Is It?

On the made-up month of 439 payments (worth about ₦37 million):

- 76% were closed without a person: 70% by the certain rules, and 6% by the model.
- **0 wrong closes** out of 334 automatic closes.
- 105 payments went to review, and the first 40 covered 91% of the money at risk.
- It would save about 17 hours of work a month.

But this data is made up. On real data, there **will** be some wrong closes. The 100% figure won't survive real life.

## 8. What's Weak or Missing

- All results come from made-up data, so they depend on the guesses used to make it.
- The 85% line was set using only 87 payments.
- Cash can't be confirmed with Paystack, so it's only "reported", not "verified".
- Nothing retrains by itself. Rejections are saved for a person to use later.
- The review list sorts by money only. Small old payments could wait forever.
- Only Paystack is connected so far.
- Without the optional AI extra installed, 8 tests fail. The project's notes say they shouldn't.
- It hadn't been deployed online yet, according to the project's notes.

## 9. What This Project Shows You Can Do

- Solve a real, everyday business problem with careful design.
- Mix certain rules, a model, and human judgement in the right order.
- Turn "how sure is sure enough?" into a money decision.
- Handle money safely: whole kobo, double-checks, permanent records.
- Be honest that results on made-up data won't fully hold in real life.

## 10. Ten Things to Remember

1. Reckon matches payments to invoices for Nigerian businesses.
2. Certain rules go first, a careful model second, and people last.
3. Every Paystack alert is checked and then confirmed directly with Paystack.
4. Money is stored as whole kobo, never decimals.
5. The rules refuse to guess when two invoices fit equally.
6. The model only closes a match when it's at least 85% sure.
7. 85% comes from costs: one mistake costs about 83 checks.
8. The review list puts the biggest money first.
9. Every decision goes into a permanent record.
10. On made-up data: 76% closed automatically, with 0 mistakes.

## Where to Go Next

- For the system explained step by step with a diagram, read `10-system-design-for-beginners.md`.
- For every technical word explained, read `11-technical-terms.md`.
- For every tool explained, read `12-tools-and-why.md`.
- For the full technical version, read `01-system-design.md` and `06-explain-to-technical.md`.
- To practise explaining it out loud, read `07-defend-in-interview.md`.
