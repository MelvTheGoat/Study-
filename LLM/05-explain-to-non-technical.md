# LLM (gptlab): Explained Simply

Repo: https://github.com/MelvTheGoat/LLM

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

I wrote, from scratch, everything needed to teach a small version of ChatGPT-style AI to write text, using free computers.

## First, what is a "language model"?

A language model is a program that has read a huge amount of text and learned to guess the next word. ChatGPT is a very large one. Mine are tiny in comparison: about 1 million to 100 million "settings" (called *parameters*, the numbers the program adjusts as it learns), against hundreds of billions for the big ones.

## The everyday comparison

Imagine teaching a child to read using a huge library, but you only get the library for 11 hours at a time. Then the doors close, and you have to come back another day.

To make that work you'd need to:
- **Bookmark exactly where you stopped**, down to the sentence.
- **Remember everything the child had learned so far**, not just the page number.
- **Keep your notes safe**, so if you get kicked out mid-sentence you don't lose yesterday's notes.
- **Keep a timetable**, so you know which lesson comes next when you return.

That's exactly the problem my project solves. The free computers I use (from a website called Kaggle) switch off after about 11 hours. My system saves its progress just before, and next time it carries on from exactly the same spot, so precisely that the results come out identical to never having stopped. I have an automatic test that proves this.

## How it works, step by step

1. **Collect reading material.** I use a free, public collection of educational web pages.
2. **Tidy it up.** Remove broken pages, repeated pages and pages that are mostly symbols, and keep a count of everything removed and why.
3. **Keep a "test pile" aside.** A small share of the pages is never used for learning, only for checking progress. It's like keeping some exam questions secret.
4. **Break text into pieces.** The model reads "tokens" (common chunks of letters) rather than whole words.
5. **Train.** The model reads, guesses the next token, checks, and adjusts, millions of times.
6. **Save progress safely** every so often, and just before the computer switches off.
7. **Test it**: how well it predicts the secret test pile, and a standard quiz where it picks the most sensible ending to a short story (guessing at random scores 25%).

## Where it stands

All the machinery is built and checked. **A first test run on the free graphics computers has now passed**, and it measured how fast training goes. The real training experiments haven't run yet, so there are no scores to share. That's the next step, and I'll only report numbers that come from real runs.

## What this shows about me

- I understand how AI text models actually work on the inside, not just how to use them.
- I can build systems that keep working on unreliable, time-limited machines.
- I plan experiments carefully and don't claim results before I have them.
- I can do serious work with zero budget.
