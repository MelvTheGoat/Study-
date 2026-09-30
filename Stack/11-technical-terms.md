# Reckon, Payment Matching: Technical Terms

This file explains every technical term used in this project, in plain English. For each one you get two things: **what it means**, and **why this project needed it**. Read it alongside [10-system-design-for-beginners.md](10-system-design-for-beginners.md).

The terms follow a payment's journey: the money basics, receiving it, matching it, checking the model, and keeping records.

---

## 1. Money and Business Basics

### Invoice
**What it means:** a bill sent to a customer, saying how much they owe and for what.

**Why it's needed here:** every payment should settle an invoice. Matching them is the whole job.

### Reconciliation
**What it means:** matching payments to invoices so the books add up.

**Why it's needed here:** it's the evening chore Reckon automates. Doing it by hand goes wrong in the same ways every time.

### Kobo (integer money)
**What it means:** the smallest unit of the naira (₦1 = 100 kobo). "Integer" means a whole number, with no decimals.

**Why it's needed here:** computers store decimals with tiny errors that add up. Storing every amount as whole kobo means sums are always exact. Fractions of a kobo cause an error.

### Dedicated virtual account
**What it means:** a bank account number given to one customer, so any money sent to it is clearly theirs.

**Why it's needed here:** a payment into a customer's own account is strong evidence. If they have exactly one invoice of that amount, the match is certain.

### Part payment and overpayment
**What it means:** paying less than the invoice (part payment) or more than it (overpayment).

**Why it's needed here:** these happen often and break simple "exact amount" matching. The model has facts for "underpaid" and "overpaid".

### Settlement and fees
**What it means:** settlement is when the payment company actually sends the money to your bank. Fees are what it keeps.

**Why it's needed here:** Paystack pays out a day later (T+1) and takes a fee. So the bank credit is smaller and later than the sales. The daily report shows sales, fees and timing separately, so there's no mystery gap.

---

## 2. Receiving Payments

### Paystack
**What it means:** a Nigerian payments company that handles card, transfer and account payments for businesses.

**Why it's needed here:** it's the payment source Reckon is wired up to.

### Webhook
**What it means:** an automatic message one service sends to another when something happens, like "payment successful".

**Why it's needed here:** Paystack tells Reckon about payments this way. Reckon handles 6 kinds of payment event.

### Signature (HMAC SHA-512)
**What it means:** a secret stamp added to a message, made with a key only the sender and receiver know. HMAC SHA-512 is the method used.

**Why it's needed here:** anyone could send a fake "payment successful" message. Checking the stamp proves it really came from Paystack.

### Verify call
**What it means:** asking Paystack directly, "Is this payment real, and how much was it?"

**Why it's needed here:** a webhook is a snapshot that could be stale or faked. Paystack's direct answer always wins.

### Idempotent
**What it means:** doing the same thing twice has the same effect as doing it once.

**Why it's needed here:** Paystack resends webhooks if a reply is slow. The same payment must never be counted twice.

### Background task
**What it means:** work done after replying, so the reply isn't slowed down.

**Why it's needed here:** Reckon replies "got it" at once, then does the slower checks. This stops Paystack from resending. The background task uses its own database connection, because of how the web tool orders its steps.

### Forward-only status
**What it means:** a payment's status can only move forward (pending, failed, success, reversed, refunded), never back.

**Why it's needed here:** messages can arrive out of order, like a refund before its charge. Moving only forward means an old message can't undo a newer state.

### Intake (prose reports)
**What it means:** reading payment reports written in everyday language, like a WhatsApp message saying "Ada paid 50k for invoice 12".

**Why it's needed here:** many Nigerian businesses report payments this way. Patterns pull out amounts, names, dates and invoice numbers, or refuse if they can't.

### LLM reader (optional)
**What it means:** an AI that reads text (a large language model), used only for reports the patterns can't read.

**Why it's needed here:** it's the last resort, and it's off by default. It must answer in a fixed format, may say "can't read", and is only asked once. That way there are no surprise bills.

### JSON schema
**What it means:** a description of exactly what fields and types a piece of data must have.

**Why it's needed here:** the LLM must fill in the same shape as the pattern reader. The schema is generated from the same code, so the two can't drift apart.

---

## 3. Matching Payments

### Deterministic rule
**What it means:** a rule with a clear yes-or-no answer every time, with no guessing.

**Why it's needed here:** certain matches (an exact reference, or the only invoice of that exact amount) close without any model.

### Tie
**What it means:** when two or more invoices fit equally well.

**Why it's needed here:** a tie isn't certainty, so the rules refuse to pick one.

### Duplicate guard
**What it means:** a check for the same payment arriving twice.

**Why it's needed here:** same money, same payer, same day, and the invoice is already paid is suspicious. A person decides.

### Name similarity
**What it means:** a score for how alike two names are, allowing for spelling mistakes and word order.

**Why it's needed here:** bank notes look like "NIP/GTB/OKONKWO ADA/PAYMENT". The matcher removes noise words like "NIP" and "PAYMENT", handles short forms and spelling variants, and avoids merging two different customers with similar names.

### Feature
**What it means:** one fact, turned into a number, that the model uses.

**Why it's needed here:** the model uses 18 features per payment and invoice pair, like "surname matches exactly", "amount is exact" and "seen this payer before". Each is something a bookkeeper would check, so the reasons can be explained in words.

### Logistic regression
**What it means:** a simple model that gives each feature a weight, adds them up, and turns the total into a probability between 0 and 1.

**Why it's needed here:** with 18 features and a few thousand examples, a simple model is enough. It's written out by hand in plain Python, so every step is readable.

### Threshold
**What it means:** the cut-off line for acting automatically.

**Why it's needed here:** the model only closes a match at 0.85 (85%) or above. The line comes from costs: a wrong close (₦5,000) costs about 83 times a review (₦60).

### Cost matrix
**What it means:** a table of what each kind of mistake or action costs.

**Why it's needed here:** it turns "how sure is sure enough?" into a business decision, written down in naira.

### Review queue
**What it means:** a list of payments waiting for a person to decide.

**Why it's needed here:** anything uncertain goes here, sorted by money at risk, with a suggested match and reasons. The first 40 items cover 91% of the money.

---

## 4. Checking the Model

### Synthetic data (corpus)
**What it means:** realistic, made-up data. A corpus is a collection of examples.

**Why it's needed here:** real business payments can't be published. The project generates a month of payments with an answer key, so every result can be checked and rebuilt.

### Answer key (ground truth)
**What it means:** the correct answer for every example.

**Why it's needed here:** it's what the matches are marked against.

### Precision and recall
**What it means:** precision is how many automatic closes were right. Recall is how many of all matches were found automatically.

**Why it's needed here:** precision matters most, because a wrong close is expensive. On the made-up data, precision was 100% (0 wrong out of 334) and recall 80%.

### Calibration
**What it means:** whether "85% sure" really means right 85% of the time.

**Why it's needed here:** the threshold only works if the probabilities are honest. The project uses Platt scaling, which reshapes the model's scores so they match real rates. It beat the other method tested.

### Brier score
**What it means:** a score for how good probabilities are, where 0 is perfect.

**Why it's needed here:** it was used to compare calibration methods. After calibration it was 0.0219 on held-out data.

### Cross-validation
**What it means:** splitting data into several parts, then testing on each part in turn while learning from the others.

**Why it's needed here:** only 87 payments reached the model stage, too few to set some aside. Cross-validation uses them all fairly to pick the threshold.

### Label
**What it means:** the correct answer attached to an example, used for learning.

**Why it's needed here:** when a person rejects a suggestion, it's saved as a label. Nothing retrains automatically, but a person can use the labels later.

---

## 5. Records and Running

### Audit log (append-only)
**What it means:** a permanent record of every decision. "Append-only" means entries can be added but never changed or deleted.

**Why it's needed here:** anyone can trace exactly why a payment was closed: which part decided, the score, the evidence, when and who.

### API
**What it means:** a way for programs to talk to each other by sending requests to web addresses.

**Why it's needed here:** Reckon offers its own small API for review actions and labels, and calls Paystack's API to verify payments.

### Smoke test
**What it means:** a quick check that an app starts and its main pages work.

**Why it's needed here:** the deploy script refuses to ship a version that fails it.

### Container
**What it means:** a sealed box holding an app and everything it needs, so it runs the same anywhere.

**Why it's needed here:** Reckon is packaged this way to run on Google Cloud Run. The project's notes say it hadn't been deployed yet.
