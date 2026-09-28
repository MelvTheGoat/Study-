# Nigerian-Fintech-Compliance-RAG: Explained Simply

Repo: https://github.com/MelvTheGoat/Nigerian-Fintech-Compliance-RAG

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A question-and-answer website for Nigerian financial technology rules. You ask in plain English, and it answers using only the official rule books, showing the exact paragraph each point came from.

## The everyday comparison

Imagine a very careful librarian. You ask, "How quickly must a payment company report a data breach?" A careless helper would answer from memory. This librarian instead:
1. finds the right pages in the rule books,
2. reads you the answer **only** from those pages,
3. puts a sticky note on each sentence saying which page it came from,
4. and if the books don't cover your question, says "I don't know" instead of making something up.

That's what my app does, using AI to write the answer but only from the pages it found.

## Why it's useful

Nigerian fintech rules are spread across different regulators (the Central Bank and the Data Protection Commission), in long documents that refer to each other. Some questions even have two correct answers, one for each regulator. Reading hundreds of pages takes hours. Guessing can be costly.

## How it works, step by step

1. **It prepares the rule books ahead of time,** cutting them into short sections and building two kinds of index: one for exact words (like "Tier 1") and one for meaning (so "report a breach" finds "notify the Commission").
2. **When you ask a question,** it first removes any personal details (like BVN or phone numbers) so they're never sent anywhere.
3. **It searches both indexes** and combines the results.
4. **It gives the best 6 passages to an AI model** with strict instructions: only use these, cite each point, and say so if they don't answer the question.
5. **It double-checks the answer.** If the AI cites a page that wasn't in the search results, that citation is removed and the sentence is marked "unverified".

## How well does it work?

On 60 test questions I wrote and checked by hand, the right passage was among the top five results **96%** of the time.

## Honest limits

- The documents in the project right now are **summaries I wrote for testing**, not the official rule books (the official websites were blocked from where I built it). The website warns about this clearly.
- It's **not legal advice**, and it can't tell you whether *your* company is compliant.

## What this shows about me

- I can build a practical AI tool that is careful about truth and sources.
- I measure quality honestly, including the things that didn't work.
- I protect personal data by design.
- I make things run cheaply (it uses about a tenth of the memory of the usual approach).
