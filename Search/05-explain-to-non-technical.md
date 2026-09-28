# Search (Job Hunt): Explained Simply

Repo: https://github.com/MelvTheGoat/Search (private)

*For a family member or a recruiter with no tech background.*

---

## What I built, in one sentence

A personal assistant program that searches hundreds of company websites every day for data and AI jobs, works out which ones I could really get from Nigeria, and ranks them by how well they match my CV.

## The everyday comparison

Imagine you're house-hunting in a new city. Every day you'd have to:
- check dozens of estate agents' windows,
- ignore the same house listed by three agents,
- skip houses that say "students only" or "no pets" in the small print,
- and sort the rest by how well they suit you.

My tool is like a friend who does that walk every morning and hands you a short, sorted list, with a note on each house saying why it's there and what the catch is.

## How it works, step by step

1. **It collects job posts** from 405 company career pages and several job websites, but only sites that allow automatic reading. It waits politely between requests so it doesn't overload anyone.
2. **It removes repeats.** The same job often appears on several sites.
3. **It keeps only relevant jobs:** data, machine learning and AI.
4. **It checks "can I get this?"** It reads the small print for phrases like "must already have the right to work in the UK". It also checks official government lists of companies allowed to sponsor work visas in the UK and the Netherlands, and past US visa filings. When a job is ruled out, it saves the exact sentence that ruled it out.
5. **It gives each job a score out of 100** for how well it matches my CV. It uses a small AI model on my own laptop (free and private) to compare the meaning of the job text with my CV and projects, plus simple checks on skills and seniority.
6. **It keeps a tracker spreadsheet** of every job, where I record "applied", "interview" and so on. It never overwrites what I've written.
7. **It helps with cover letters, but checks them.** The letters are drafted with an AI assistant. Then the tool checks every number in each letter against my CV, so a letter can never claim something I didn't do.

## What it never does

It never applies for me. I read every job and apply myself.

## What this shows about me

- I turn a messy, repetitive task into a reliable daily system.
- I care about the details that matter in real life (visas and work rights), not just matching keywords.
- I use AI carefully: free, private, and checked.
- I respect website rules. I don't scrape sites that forbid it.
