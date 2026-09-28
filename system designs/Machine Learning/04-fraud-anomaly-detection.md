# Fraud and Anomaly Detection

A fraud detection system checks every payment and flags the ones that look suspicious. If your bank has ever texted you "Did you just spend £500 in another country?", that was a fraud system at work. It matters because fraud costs people and companies real money, but blocking honest customers by mistake is costly too.

## Key Terms

- **Transaction**: one payment, like buying a coffee with your card.
- **Fraud**: a payment made by someone who shouldn't be making it, like a thief using a stolen card.
- **Anomaly**: something unusual that doesn't fit a normal pattern.
- **Model**: a program that learned patterns from past data.
- **Label**: the confirmed answer for a past transaction, "fraud" or "not fraud".
- **False positive**: an honest payment we wrongly flag as fraud.
- **Precision and recall**: two ways to measure how well we catch fraud (explained with numbers below).

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** We want to stop as much fraud as possible without annoying honest customers. These two goals pull against each other, so we need to decide how to balance them.

**Step 2: Figure out the data.** We have each payment's details plus the customer's history. We also have labels, but they arrive late: a customer might report fraud weeks after it happened.

**Step 3: Sketch the main parts.** A typical design mixes simple rules (for obvious cases) with a model (for subtle ones). Then a decision step chooses: approve, review, or block.

**Step 4: Walk through one payment.** The card is being swiped right now, so the check must be fast, well under a second. Follow one payment through each box.

**Step 5: Decide how to know it works.** Count how much fraud we catch, how many honest people we bother, and how much money is saved.

**Step 6: Plan for problems.** Fraudsters change tactics once they're blocked. The system needs a way to learn and adapt continuously.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

Let's design fraud checks for a card payments company.

- **1 million** transactions per day.
- Decide within **100 milliseconds** (0.1 seconds) per payment.
- About **1 in 1,000** transactions is fraud.

How much fraud is that? Step by step:

1. 1,000,000 transactions per day.
2. ÷ 1,000 (one fraud in every thousand).
3. = **about 1,000 fraudulent payments per day**, hidden among 999,000 honest ones.

That's like finding 1 bad apple in a crate of 1,000 good ones, very quickly.

### The Data We Use (Step 2)

- **Payment details**: amount, shop, country, time, device.
- **Customer history**: usual spending, usual countries, number of payments in the last hour.
- **Labels**: which past payments turned out to be fraud, usually learned from customer complaints.

The history is the key. £500 is normal for some people and very strange for others. Fraud is often about *change*, not the payment by itself.

### The Big Picture (Step 3)

```
   [Card payment arrives]
            |
            v
   [Quick Rules]      -- obvious cases, e.g. stolen-card list
            |
            v
   [Fraud Model] <---- [Customer History]
            |
            v
   [Decision: approve / review / block]
            |
            v
   [Human Review Team] ----> [Labels] -> retrain model
```

Step by step:

1. A card payment arrives and must be decided within 0.1 seconds.
2. Quick Rules catch obvious cases, like a card already reported stolen.
3. The Fraud Model looks at the payment plus the Customer History and gives a risk score from 0 to 100.
4. The Decision step approves low scores, blocks very high scores, and sends the middle ones for review.
5. The Human Review Team checks unclear cases. Their answers become labels that help retrain the model.

### The Main Parts (Step 3)

**Quick Rules.** These are simple "if this, then that" checks, like "block cards on the stolen list". They're like a security guard with a list of known troublemakers: fast and clear, but they only catch what's on the list.

**Customer History.** This is a store of ready-made facts, like "average spend: £40" or "payments in the last hour: 1". It's like a doctor reading your medical notes before judging whether a symptom is worrying. Keeping these facts ready makes the check fast.

**Fraud Model.** The model learned from millions of past payments which patterns tend to mean fraud. For example, a sudden burst of small payments at 3 a.m. from a new phone. It's like an experienced bank clerk who gets a "something feels off" instinct, except it checks every payment, every time.

Some systems also look for anomalies without any labels, by flagging anything that looks very different from normal (advanced - skip for now).

**Decision Step.** This turns the score into an action using two thresholds (cut-off points). For example: under 30 = approve, 30 to 90 = review, over 90 = block. It's like traffic lights: green, amber, red.

**Human Review and Labels.** Trained staff check the amber cases, often by texting the customer. Their answers, plus customer complaints, become labels. The model is retrained regularly with these labels, so it keeps learning.

### How We Know It's Working (Step 5)

Imagine one day with 1,000 real frauds. The model flags 2,000 payments, and 800 of them really are fraud.

- **Recall** = frauds we caught ÷ all frauds = 800 ÷ 1,000 = **80%**. Think of a fishing net: recall is how many of the fish you actually caught.
- **Precision** = real frauds ÷ everything we flagged = 800 ÷ 2,000 = **40%**. This is how much of your net is fish rather than old boots.
- **False positives** = 2,000 − 800 = **1,200** honest customers bothered. Too many, and people stop trusting their card.
- **Money saved**: the value of fraud blocked, minus the cost of reviews and upset customers.

Moving the thresholds changes the balance. A stricter system catches more fraud but bothers more honest people.

### What Can Go Wrong (Step 6)

- **Fraudsters adapt.** Once a trick is blocked, they try a new one. Retrain often and watch for sudden changes in fraud patterns.
- **Labels arrive late.** Some fraud is only reported weeks later. Remember that recent numbers look better than they really are until all the labels come in.
- **Too many false alarms.** Honest customers get annoyed. Tune the thresholds and track complaints.
- **The model is down.** Payments can't wait. Fall back to the Quick Rules so payments keep flowing while you fix it.

## Quick Recap

- Fraud is rare (about 1 in 1,000), so the system is looking for needles in a haystack.
- Combine quick rules for obvious cases with a model for subtle ones.
- Use three outcomes (approve, review, block), not just yes or no.
- Measure recall (fraud caught) and precision (flags that were right), and watch false alarms.
- Fraudsters change tactics, so keep learning from new labels.
