# Fraud-Detection-With-Sequence-Models: Explained Simply

Repo: https://github.com/MelvTheGoat/Fraud-Detection-With-Sequence-Models

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A system that checks every card payment in real time and flags likely fraud, plus a fair test of whether a newer kind of AI that looks at the *order* of your recent payments does better than the standard approach.

## The everyday comparison

Imagine two security guards at a shop.
- **Guard A** has a checklist: "Is this purchase unusually big? Is it from a new phone? Has this card been used many times today?" Guard A is fast and very good.
- **Guard B** watches the *story* of each customer: "This person usually shops in the afternoon, then suddenly nine tiny purchases in four minutes from a new phone. That's odd for *them*."

I tested both guards on the same made-up shop for six months, with three known kinds of fraud. Then, near the end, **the criminals changed tactics** to dodge the checks.

**What happened:**
- Before the criminals changed tactics, both guards were **equally good**. Guard A (the standard approach) was even slightly ahead.
- After they changed tactics, **Guard B held up much better**. Guard A relied on exactly the clues the criminals learned to hide.
- But when I counted the **money saved**, the difference was too small to be sure about.

## How it works, step by step

1. **Make realistic practice data:** hundreds of thousands of pretend payments, including tricky normal behaviour (people travelling, buying in bursts, using a new phone).
2. **Build features carefully,** only ever looking at *earlier* payments, never "peeking into the future".
3. **Train both kinds of model,** three times each, to check the results aren't luck.
4. **Choose when to block a payment** by weighing costs: annoying a genuine customer vs losing money to fraud.
5. **Run it live:** each payment is checked in about 7 thousandths of a second.
6. **Watch for criminals changing tactics** with alarms that react quickly.

## Honest findings

- The newer AI is better at **spotting patterns** and **coping when criminals adapt**.
- It **didn't clearly save more money** in my tests. More testing would be needed.
- The **simplest** of the newer models won, not the most complicated one.
- An earlier version of my write-up claimed a bigger saving. I found my mistake and corrected it.

## What this shows about me

- I test ideas fairly instead of assuming the fancy option wins.
- I understand how fraud really works and how criminals adapt.
- I build systems that are fast enough for real payments.
- I'm honest when results are mixed.
