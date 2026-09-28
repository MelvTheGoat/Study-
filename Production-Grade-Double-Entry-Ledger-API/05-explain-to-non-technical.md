# Production-Grade-Double-Entry-Ledger-API: Explained Simply

Repo: https://github.com/MelvTheGoat/Production-Grade-Double-Entry-Ledger-API

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

The behind-the-scenes "money book" for a payments app: the part that records every payment, keeps every balance correct, and makes sure money is never lost, duplicated or secretly changed, even when lots of things happen at once.

## The everyday comparison

Imagine an old-fashioned bank book, written in ink.
- Every payment is written as **two lines**: money leaves one account and arrives in another. The two lines always add up to zero, so money is never created or destroyed. That's called **double-entry** bookkeeping, and accountants have used it for centuries.
- You **never rub anything out**. If there's a mistake, you write a new line that reverses it. So the history is always there.

My system is that bank book, built so that even the computer itself **refuses** to break those rules.

## The tricky situations it handles

1. **Two payments at the same moment.** Alice has £100. Two people try to take £60 from her at exactly the same time. A careless system lets both through and Alice ends up at −£20. Mine makes them queue, one after the other, so only one succeeds.
2. **"Did my payment go through?"** Your phone loses signal after you press "Pay". Did it work? You press again. A careless system charges you twice. Mine recognises the second press as a repeat and just shows you the first result.
3. **Telling other systems.** After a payment, other parts of the business (like a shop releasing goods) need to know. My system writes the "tell others" note in the same step as the payment, so they never hear about a payment that didn't happen, and never miss one that did.

## How do I know it works?

- **140 automatic tests** run against a real database. When I ran them, they all passed.
- One test fires dozens of payments at the same account at the same instant and checks that **exactly** the right number succeed and the balance is **exactly** right.
- A stress test pushed through **17,697 payments**, and afterwards the books balanced **exactly** to zero.
- I measured two different ways of handling "at the same moment" payments. The approach I chose handled about **3.5 times more** payments per second on a busy account.

## What this shows about me

- I understand how money systems must work: accuracy, history and safety.
- I can build software that stays correct when many things happen at once.
- I prove claims with tests and measurements, and I report the weak spots too.
