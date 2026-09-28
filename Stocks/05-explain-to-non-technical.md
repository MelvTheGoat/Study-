# Stocks: Explained Simply

Repo: https://github.com/MelvTheGoat/Stocks

*For a family member or a recruiter with no tech background.*

---

## What I'm building, in one sentence

A computer assistant that answers factual questions about company shares, like "what was this company's share price on this day?", plus a fair test that checks how often it gets the answer right.

## The everyday comparison

Think about hiring a new accountant.

You wouldn't hire them because they *sound* confident. You'd give them a test: ten questions where you already know the answers, marked the same way for every candidate.

AI chat tools are like a very confident new hire. They speak well, but they sometimes make numbers up. My project writes **the test first**, then builds the assistant, then marks it honestly.

I'm also testing it on two kinds of companies:
- **American companies** like Apple, which AI tools have read a lot about.
- **Nigerian companies** on the Nigerian stock exchange, which AI tools know much less about.

It's like testing someone on their home town and then on a town they've never visited. The difference shows how much they're *remembering* versus actually *working things out*.

## What's done so far

This project is at an early stage. What's finished is the groundwork:

1. **A settings file for every test run.** Each test is described in one small file. If there's a typo in it, the system refuses to run instead of quietly ignoring it. That keeps the results honest.
2. **A careful way to talk to the AI.** It saves every answer, so repeating a test costs nothing. If the AI's computer is briefly busy it waits and tries again, but it doesn't keep retrying a question that can never work. It also keeps a record of every question asked and how long it took.
3. **Automatic checks.** 64 small automatic tests make sure all of this works. They even run with the internet switched off, to prove nothing secretly depends on an outside website.

## What's not done yet

- The **question-and-answer test** itself.
- The **assistant** that answers questions.
- The **share-price data**.

There are no results yet, and I don't pretend there are.

## The tricky part

Getting Nigerian share prices legally is harder than expected. The Nigerian stock exchange's website blocks automatic programs, and I won't sneak past that barrier. So I'm writing down every possible data source, checking its rules first, and only using sources that allow it.

## What this shows about me

- I care about **proving** things work, not just showing a nice demo.
- I think about **fairness and honesty** in testing.
- I respect **data rules** even when it slows me down.
- I build solid **groundwork** before the flashy part.
