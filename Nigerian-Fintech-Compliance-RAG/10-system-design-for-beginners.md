# Nigerian Fintech Rules Q&A: System Design for Beginners

This project is a question-and-answer app for Nigerian fintech rules, like data protection and customer checks. It answers in plain English using only its stored documents, cites every claim, shows the exact passage underneath, and refuses when the documents don't cover the question. One important note: it currently holds six written summaries, not the official regulations, so it's a working demo, not a legal tool.

## Key Terms

- **LLM (large language model)**: an AI that reads and writes text, like ChatGPT.
- **RAG (retrieval-augmented generation)**: first *find* the right passages, then have the LLM write an answer using only those. Like an open-book exam.
- **Chunk**: a small piece of a document, here usually one section of a rule.
- **Embedding**: a list of numbers that captures what text means. Similar meanings get similar numbers.
- **Keyword search**: finding passages that contain the same words as the question.
- **Citation**: a label showing which passage a claim came from, like [S1].
- **PII (personally identifiable information)**: details that identify a person, like a phone number or BVN.

---

## Part 1: How to Approach It

**Step 1: Understand the goal.** Wrong legal answers are dangerous. So the goal is: only answer from the documents, show proof, and refuse when unsure.

**Step 2: Figure out the data.** The rules come from the Central Bank of Nigeria (CBN), the Nigeria Data Protection Act and the data protection commission (NDPC). The official PDFs couldn't be downloaded when this was built, so six summaries stand in for now.

**Step 3: Sketch the main parts.** Documents are split and indexed once, ahead of time. Then each question goes through a safety check, two kinds of search, the LLM, and a final check.

**Step 4: Walk through one question.** Follow "How fast must we report a data breach?" from the question box to a cited answer.

**Step 5: Decide how to know it works.** Test whether search finds the right passage, and whether answers stick to the passages.

**Step 6: Plan for problems.** The LLM might invent a citation, a user might paste personal data, and free services have limits. Plan for each.

---

## Part 2: The Design

### What It Needs to Do (Step 1)

- Search **6** documents, split into **100** chunks.
- Use at most **2 GB** of memory, so it fits on a small free server.
- Allow **8** questions per 5 minutes per visitor, and **300** a day overall.
- Remove personal details **before** anything is sent to an outside AI service.

What happens to a long section? Step by step:

1. Each chunk holds up to **1,200** characters. Short sections stay whole.
2. When a section is too long, it's split, and each piece repeats the last **180** characters of the one before. So a sentence at the edge still appears whole somewhere.
3. Each new piece moves forward by up to 1,200 − 180 = **1,020** characters.

### The Big Picture (Step 3)

```
   [Question]
       |
       v
   [Input Guard: remove personal data, check limits]
       |                     |
       v                     v
   [Keyword Search]    [Meaning Search]
       |                     |
       +---------+-----------+
                 v
   [Merge the two rankings: top 6 passages]
                 |
                 v
   [LLM writes answer with citations]
                 |
                 v
   [Output Guard] ---> [Answer + source passages]
```

Step by step:

1. A user asks a question.
2. The Input Guard checks its length, looks for tricks, removes personal details like BVNs and phone numbers, and checks the limits.
3. Keyword Search and Meaning Search each rank all 100 chunks.
4. The two rankings are merged, and the top 6 passages are labelled S1 to S6.
5. The LLM is told: answer only from these passages, cite each claim, and say so if they don't cover it.
6. The Output Guard removes any citation to a passage that wasn't given, and marks that claim "unverified".
7. The user sees the answer, then every cited passage in full.

### The Main Parts (Step 3)

**Chunking by Section.** Legal text is organised by numbered sections, so each chunk is usually one section. That way a citation points to one clear clause. It's like cutting a textbook at chapter headings, not at random pages.

**Keyword Search.** It finds chunks with the same words as the question. Rules use exact terms like "Tier 1" or "72 hours", so exact matching really helps. It's like using a book's index.

**Meaning Search.** It compares embeddings to find chunks with a similar *meaning*, even with different words. The model runs with ONNX Runtime (a light tool for running AI models), which uses far less memory than the usual tools. It's like asking a librarian who understands what you mean.

**Merging the Rankings.** The two searches score in different ways, so their scores can't be compared directly. Instead, their *positions* are combined: a chunk ranked high in both wins. It's like two judges each ranking contestants, then adding up the places.

**The LLM and Its Rules.** The LLM only sees the 6 passages, and must cite them. If they don't cover the question, it must use a special "refuse" signal. The app can use a free AI service, or a "stub" that just quotes passages, which is used for testing.

**Input and Output Guards.** The input guard strips personal details before they reach an outside service. The output guard catches invented citations and makes them visible. They're like security at the entrance and exit of a building.

Re-ranking the shortlist with a second, more careful model is possible but switched off (advanced - skip for now).

### How We Know It's Working (Step 5)

- **Recall@5**: how often the right passage is in the top 5 results. It's **96%** on 50 test questions.
- **MRR**: how high the first right passage appears, on average (1.0 means always first). It's **0.84**.
- **Correct refusals**: 10 off-topic questions should be refused. The testing stub only refused 40%, and a real LLM hasn't been measured yet.
- **Memory**: peak about **225 MB**, well under the 2 GB limit.

### What Can Go Wrong (Step 6)

- **The documents are summaries, not the real rules.** Some limits are out of date. The app shows a warning, and a file lists the official PDFs to add.
- **The LLM invents a citation.** The output guard removes it and marks the claim "unverified".
- **Off-topic questions slip through.** Search scores for off-topic and fair questions overlap too much to filter by score. So refusing is left to the LLM.
- **Short forms confuse search.** "CISO" might not match "Chief Information Security Officer" in the text.

## Quick Recap

- RAG means: find the right passages first, then answer only from them.
- Two searches, by keyword and by meaning, are merged by rank.
- Every claim is cited, and invented citations are caught.
- Personal details are removed before anything leaves the app.
- The documents are sample summaries for now, not the official regulations.
