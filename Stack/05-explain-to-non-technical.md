# Stack ("Reckon"): Explained Simply

Repo: https://github.com/MelvTheGoat/Stack

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A program that works out which bill each incoming payment is for, ticks off the ones it's sure about, and gives a person a short, sorted list of the tricky ones.

## The everyday comparison

Imagine running a shop where customers pay in four ways: card, their own special account number, a bank transfer with just a name on it, or cash. At the end of the day you have a pile of payments and a pile of bills, and you have to pair them up.

Some are easy: the card payment has the bill number on it. Some are hard: "OKONKWO ADA PAYMENT" for ₦45,000, when you have two customers called Okonkwo and three bills for ₦45,000.

My program is like a careful assistant who:
1. **pairs up the obvious ones first**, the ones where there's no doubt at all,
2. **makes an educated guess on the next group**, but only acts on a guess if it's almost certainly right,
3. **puts everything else on your desk**, with the biggest amounts on top and a note saying "I think this is Ada's bill 42, because the name and amount match".

## Why "almost certainly right" matters

Getting a match wrong is expensive. You might chase a customer for money they already paid. Double-checking a match is cheap: a couple of minutes. I worked out the costs in naira: a wrong match costs about 83 times more than a check. So the program only acts alone when it's more than about 99% sure.

## How well it works

I tested it on a realistic, made-up month of 439 payments worth about ₦37 million (real customer data can't be shared):
- About **three quarters** were matched with no one needing to look.
- **None** of those were matched wrongly.
- The rest went to a person, and **the first 40 on the list covered 91% of the money** that needed checking.
- That saves about **17 hours of work a month**.

Because the test data is made up, real life will be messier, and I say that openly.

## Safety

- It **never moves money**. It only reads and suggests.
- Every decision is written down permanently: what it decided, why and when.
- Money amounts are stored exactly, down to the last kobo, so nothing drifts.

## What this shows about me

- I understand how payments really work in Nigeria.
- I use AI and statistics carefully, only where they're trusted, and I measure them.
- I think about business costs, not just technical scores.
- I'm honest about limits.
